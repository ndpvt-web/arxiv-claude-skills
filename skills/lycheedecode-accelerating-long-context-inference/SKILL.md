---
name: "lycheedecode-accelerating-long-context-inference"
description: "Accelerate long-context LLM inference using hybrid-head sparse decoding with HardKuma-based head partitioning. Implement KV cache optimization that classifies attention heads into retrieval heads (full attention + top-k selection) and sparse heads (reuse selected tokens). Triggers: 'optimize long-context inference', 'reduce KV cache memory', 'speed up decoding for long sequences', 'implement sparse attention decoding', 'hybrid-head attention optimization', 'accelerate 128K context generation'"
---

# LycheeDecode: Hybrid-Head Sparse Decoding for Long-Context LLM Inference

This skill enables Claude to implement and advise on LycheeDecode, a sparse decoding technique that accelerates long-context LLM inference by partitioning attention heads into two functional classes: a small set of **retrieval heads** that perform full attention and dynamically select the most important tokens, and a majority of **sparse heads** that restrict computation to only those selected tokens. This achieves up to 2.7x speedup at 128K context length while maintaining (or exceeding) full-attention quality, by preserving the functional diversity of attention heads rather than forcing uniform token sharing across all layers.

## When to Use

- When the user wants to reduce decoding latency for LLMs processing sequences longer than 4K tokens
- When implementing KV cache optimization for autoregressive generation and the user needs head-level granularity rather than layer-level sharing
- When the user asks how to classify attention heads as retrieval vs. streaming/sparse for inference acceleration
- When building custom CUDA/Triton kernels for sparse attention with workload-pooling across heterogeneous head types
- When the user wants to train a lightweight head-classification module (HardKuma gate) on top of an existing pretrained LLM
- When comparing sparse decoding strategies (Quest, TidalDecode, DuoAttention, RazorAttention) and choosing the right one for a deployment scenario
- When the user needs to optimize time-per-output-token (TPOT) in batch serving of long-context workloads

## Key Technique

**The core insight**: Not all attention heads serve the same function during decoding. Some heads ("retrieval heads") actively search the full context to find the most relevant tokens for the current generation step, while most heads ("sparse heads") attend to a narrow, relatively stable subset of positions. Prior methods like TidalDecode share a single set of important tokens across all heads in a layer, which destroys this functional diversity and degrades quality. LycheeDecode preserves it by making selection decisions at the individual head level.

**HardKuma-based head partitioning**: A Hardened Kumaraswamy distribution learns a binary gate z for each attention head. During a short training phase (~3000 steps on a single A100), the gate parameters (alpha, beta) are optimized via a combined distillation loss and L0 sparsity constraint using Lagrangian relaxation. At inference time, the gate becomes deterministic: heads with E[z] > 0.5 are classified as retrieval heads; the rest become sparse heads. Layer 0 is always treated as all-retrieval (full attention) to establish the initial token importance set.

**Decoding flow**: For each layer beyond layer 0, retrieval heads compute full attention over the entire KV cache and then run argTopK to select the k most-attended token positions. These indices are passed forward to the sparse heads in the same and subsequent layers, which compute attention only over the selected subset. This means sparse heads load a fraction of the KV cache from HBM, dramatically reducing memory bandwidth pressure. A custom TileLang kernel uses workload pooling to balance GPU thread utilization across heads with different compute loads, avoiding the idle-thread problem inherent in naive sparse/dense mixed execution.

## Step-by-Step Workflow

### A. Implementing Head Classification (Training the HardKuma Gate)

1. **Select a calibration dataset** appropriate to the target domain. Use ~512 samples of long-context text (e.g., from LongBench or domain-specific documents). The gate training is lightweight and domain-agnostic, so a general corpus works well.

2. **Initialize HardKuma parameters** for every attention head in layers 1 through L-1 (skip layer 0, which is always full-attention). Set alpha=1, beta=1 (uniform prior), stretching interval (-0.1, 1.1), learning rate 0.01.

3. **For GQA (Grouped Query Attention) models**, reduce the query dimension to match the KV head count via average pooling over the query heads within each group before computing the gate's distillation loss. This is critical for Llama3/Qwen3-style architectures.

4. **Train with dual-objective loss**: minimize KL divergence between the gated hybrid attention output and the full-attention teacher output, subject to a sparsity constraint that targets a specific number of retrieval heads N_target. Use Lagrangian relaxation: `min_{alpha,beta} max_{lambda} L_distill + lambda * (E[||z||_0] - N_target)`. The expected L0 norm has a closed-form expression under the HardKuma distribution.

5. **After training, binarize the gates**: for each head h in layer l, if E[z_h^(l)] > 0.5, mark it as a retrieval head; otherwise mark it as a sparse head. Export this classification as a binary mask alongside the model weights.

6. **Reorder weight matrices** so that retrieval heads and sparse heads form contiguous groups within each layer's Q/K/V projection. This enables the custom kernel to dispatch retrieval-head work and sparse-head work to separate thread pools without scatter/gather overhead.

### B. Deploying Sparse Decoding at Inference Time

7. **At each decoding step**, process layer 0 with full attention across all heads. From the resulting attention maps, compute `S_h^(1) = argsTopK(A_h^(0), k)` to produce the initial per-head token index sets for layer 1.

8. **For layers 1 through L-1**, execute the hybrid kernel: retrieval heads compute full softmax attention and produce new top-k index sets for the next layer; sparse heads restrict attention to the inherited index set `S_h^(l)` and propagate it unchanged as `S_h^(l+1) = S_h^(l)`.

9. **Set the token budget k** based on deployment constraints. A budget of 1024-4096 tokens works well for 128K contexts. The budget is constant across layers. At 4096 tokens on Llama3-8B, LycheeDecode matches or exceeds full-attention quality on LongBench (33.07 vs 32.33 average).

10. **For batch serving**, the workload-pooling kernel aggregates block computations from all heads into a unified work pool, partitions them into uniform splits, and distributes across GPU thread blocks. This prevents idle threads when sparse heads finish before retrieval heads in the same layer.

## Concrete Examples

**Example 1: Adding LycheeDecode-style sparse decoding to a HuggingFace model**

User: "I have a Llama3-8B model and I'm getting slow generation on 64K-token inputs. How do I implement LycheeDecode-style sparse decoding?"

Approach:
1. Profile the model to confirm decoding is memory-bandwidth-bound (check TPOT vs prefill time).
2. Run the HardKuma gate training on a calibration set (~3000 steps, single GPU).
3. Export the per-head retrieval/sparse classification mask.
4. Modify the attention forward pass:

```python
# Pseudocode for hybrid-head attention in a single layer
def hybrid_attention(q, k_cache, v_cache, head_mask, inherited_indices, top_k):
    outputs = []
    next_indices = {}

    for h in range(num_heads):
        if head_mask[h] == "retrieval":
            # Full attention over entire KV cache
            attn_weights = softmax(q[h] @ k_cache[h].T / sqrt(d_k))
            out = attn_weights @ v_cache[h]
            # Select top-k token positions for next layer
            next_indices[h] = torch.topk(attn_weights, top_k).indices
        else:
            # Sparse attention over inherited token subset only
            idx = inherited_indices[h]
            k_sub = k_cache[h][idx]
            v_sub = v_cache[h][idx]
            attn_weights = softmax(q[h] @ k_sub.T / sqrt(d_k))
            out = attn_weights @ v_sub
            next_indices[h] = idx  # propagate unchanged

        outputs.append(out)

    return torch.cat(outputs, dim=-1), next_indices
```

5. Set `top_k=2048` for 64K context (approximately 3% of sequence length).
6. Benchmark TPOT improvement. Expect 1.5-2x speedup at 64K.

Output: A modified model that generates with the same quality but reduced per-token latency, without retraining the base LLM weights.

---

**Example 2: Choosing between sparse decoding strategies**

User: "I'm comparing Quest, TidalDecode, and LycheeDecode for our 128K serving pipeline. Which should I pick?"

Approach:
1. Characterize the workload: batch size, sequence lengths, quality requirements.
2. Compare the methods on key axes:

```
| Method        | Granularity  | Selection    | Quality at 128K      | Speedup |
|---------------|------------- |------------- |----------------------|---------|
| Quest         | Page-level   | Per-head     | Degrades noticeably  | ~1.5x   |
| TidalDecode   | Layer-level  | Shared heads | Moderate degradation | ~2x     |
| LycheeDecode  | Head-level   | Hybrid       | Matches/exceeds full | ~2.7x   |
```

3. Key differentiators:
   - Quest uses page-level token selection which is coarse and misses fine-grained importance.
   - TidalDecode shares crucial tokens across all heads in a layer, destroying head functional diversity.
   - LycheeDecode preserves per-head diversity: retrieval heads find their own crucial tokens, sparse heads reuse them efficiently.
4. Recommend LycheeDecode when quality on reasoning tasks (math, code) is non-negotiable, as it scored 46.7% on AIME24 vs 40% for full attention (with cache correction).

---

**Example 3: Implementing the workload-pooling kernel**

User: "I need to write a Triton/TileLang kernel that handles mixed retrieval and sparse heads efficiently."

Approach:
1. Define the two work types: retrieval heads need full-sequence attention + top-k; sparse heads need subset attention only.
2. Instead of assigning one thread block per head (which wastes resources on sparse heads), pool all block-level work units:

```python
# Kernel design sketch (TileLang / Triton style)
# 1. Compute total work units
retrieval_blocks = num_retrieval_heads * ceil(seq_len / BLOCK_SIZE)
sparse_blocks = num_sparse_heads * ceil(budget_k / BLOCK_SIZE)
total_blocks = retrieval_blocks + sparse_blocks

# 2. Partition into uniform splits across GPU SMs
splits = partition_uniform(total_blocks, num_sms)

# 3. Each thread block processes its assigned split
#    - Determines whether this block serves a retrieval or sparse head
#    - Uses online softmax for numerically stable partial accumulation
#    - Retrieval blocks additionally write top-k indices to shared buffer

# 4. Final reduction: combine partial outputs and log-sum-exp values
```

3. Use the online softmax algorithm: maintain running max and log-sum-exp accumulators across blocks, compute partial outputs, then combine in a single final reduction pass.
4. Ensure retrieval heads write their top-k indices to a shared buffer that sparse heads in the next layer can read from.

## Best Practices

- **Do**: Always keep layer 0 as full-attention. It establishes the initial token importance set that all subsequent layers depend on.
- **Do**: Set the token budget k as a fixed absolute number (e.g., 2048 or 4096), not a percentage. The paper shows fixed budgets generalize better across sequence lengths.
- **Do**: Reorder Q/K/V weight matrices to group retrieval and sparse heads contiguously. This is essential for kernel efficiency -- scatter/gather across non-contiguous heads kills throughput.
- **Do**: Use the HardKuma training procedure rather than heuristic head classification. Manual classification (e.g., based on attention entropy) is unreliable across layers and models.
- **Avoid**: Sharing a single top-k set across all heads in a layer. This is exactly the mistake of layer-level methods like TidalDecode that degrades quality.
- **Avoid**: Setting the retrieval head count too high. The paper uses ~32 retrieval heads total (equivalent to 2 full-attention layers in TidalDecode's budget). More retrieval heads reduce the speedup with diminishing quality returns.

## Error Handling

- **Quality degradation on short sequences (<4K)**: At short contexts, the overhead of head classification and top-k selection may not pay off. Fall back to standard full attention when sequence length is below a threshold (e.g., 4096 tokens).
- **Top-k index staleness**: If the model has very deep attention patterns that shift drastically between layers, inherited indices from retrieval heads may become stale. Monitor per-layer attention overlap between retrieval-selected and oracle top-k. If overlap drops below 80%, consider adding more retrieval head checkpoints.
- **GQA head-count mismatch**: When applying to GQA models, remember that the gate operates at the KV-head level, not the query-head level. Average-pool query heads within each group before computing the distillation loss. Failing to do this will produce incorrect gate assignments.
- **Kernel numerical stability**: The online softmax accumulation across blocks requires careful handling of the log-sum-exp combination. Always use the numerically stable formulation: `output = sum(partial_out * exp(partial_lse - global_lse))`.

## Limitations

- **Not yet integrated with vLLM or TensorRT-LLM**: The current implementation uses a standalone TileLang kernel. Deploying in production serving frameworks requires custom attention backend integration.
- **Gate training is model-specific**: The HardKuma classification must be retrained for each model architecture and size. A gate trained on Llama3-8B does not transfer to Llama3-70B or Qwen3-8B.
- **Prefill phase is unaffected**: LycheeDecode only accelerates the autoregressive decoding phase. The initial prefill (processing the full prompt) still requires full attention. Combine with prefill optimization techniques (e.g., chunked prefill, prefix caching) for end-to-end gains.
- **Fixed budget across layers**: The current design uses a uniform token budget k for all layers. Adaptive per-layer budgets could improve quality-efficiency tradeoffs but are not explored in the paper.
- **Batch size sensitivity**: Speedup gains are most pronounced at smaller batch sizes where decoding is memory-bandwidth-bound. At very large batch sizes where compute becomes the bottleneck, the relative advantage diminishes.

## Reference

**Paper**: [LycheeDecode: Accelerating Long-Context LLM Inference via Hybrid-Head Sparse Decoding](https://arxiv.org/abs/2602.04541v1) (ICLR 2026). Focus on Section 3 (HardKuma gate mechanism and hybrid-head classification), Algorithm 1 (decoding procedure), and Algorithm 2 (workload-pooling kernel design). Code: [HITsz-TMG/TMGNLP](https://github.com/HITsz-TMG/TMGNLP).