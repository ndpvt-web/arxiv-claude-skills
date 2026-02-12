---
name: "meki-memory-based-expert-knowledge"
description: "Implement MeKi-style memory-based expert knowledge injection for efficient LLM scaling on edge devices. Offloads per-token expert knowledge to ROM/flash storage via static lookup tables, avoiding compute overhead. Use when: 'optimize LLM for mobile deployment', 'add storage-based scaling to a model', 'implement token-level memory experts', 'replace MoE routing with deterministic lookup', 'reduce inference latency on edge devices', 're-parameterize expert projections into lookup tables'."
---

# MeKi: Memory-based Expert Knowledge Injection

This skill teaches Claude to apply the MeKi architecture — a storage-based scaling strategy for LLMs that injects pre-stored semantic knowledge into each Transformer layer via token-indexed memory lookup tables. Instead of scaling model size (more parameters in RAM) or inference cost (more compute at test time), MeKi scales through cheap, abundant flash/ROM storage. Each layer maintains a static embedding table keyed by token ID; at inference time, knowledge retrieval is a single embedding lookup with zero matrix multiplications, enabling deployment on edge devices like smartphones with no latency overhead.

## When to Use

- When the user wants to deploy a performant LLM on mobile or edge hardware with constrained RAM and NPU budgets
- When the user asks how to scale model quality without increasing inference FLOPs or latency
- When the user needs to replace a Mixture-of-Experts (MoE) layer with a deterministic, latency-stable alternative
- When the user is building an on-device LLM pipeline and wants to leverage fast flash storage (UFS-4.0, NVMe) for model capacity
- When the user asks to re-parameterize dynamic projections into static lookup tables for inference optimization
- When the user wants to add a parallel knowledge-injection branch alongside the FFN in a Transformer

## Key Technique

**Core idea:** Attach a lightweight memory expert module in parallel with the FFN at every Transformer layer. During training, each module learns a per-token expert vector through two paths — a static embedding lookup (`M^l[token_id]`) and a dynamic SwiGLU projection of the global word embedding — fused with learnable scalars and RMSNorm. A low-rank gating mechanism, conditioned on the hidden state, modulates the expert vector before projecting it back to the model dimension. The output is added to the FFN residual stream.

**The re-parameterization trick:** The dynamic projection path (`G^l(E_global[token_id])`) depends only on the fixed global embedding table, not on runtime hidden states. After training, the dynamic path can be pre-computed for every token in the vocabulary and folded into the static table: `M_tilde^l[t] = alpha^l * RMSNorm(M^l[t] + beta^l * G^l(E_global[t]))`. This collapses two matrix multiplications and a non-linearity into a single embedding lookup at inference time. The resulting table is stored in ROM/flash — read-only, sequentially accessed, and prefetchable.

**Why this beats MoE on edge devices:** MoE routes tokens to different expert weight matrices, causing irregular memory access patterns, memory fragmentation, and unpredictable latency on mobile NPUs. MeKi's deterministic token-ID indexing enables sequential reads and asynchronous prefetching. On a Snapdragon 8 Elite, MeKi-1.7B matches 4B dense model quality at 13.7-19.9 tokens/second with zero additional latency from the memory expert branch.

## Step-by-Step Workflow

1. **Audit the target model architecture.** Identify each Transformer layer's FFN block. Confirm the model uses a standard residual structure (`h = h + FFN(norm(h))`). Record the hidden dimension `d_model`, vocabulary size `|V|`, and number of layers `L`.

2. **Choose the memory dimension `d_mem`.** This controls the capacity/storage tradeoff. The paper tests 64 to 448; 256 is a strong default. Total ROM footprint is `L * |V| * d_mem * 2 bytes` (float16). For a 28-layer model with 128k vocab and d_mem=256, that is ~1.8 GB.

3. **Implement the memory expert module for training.** For each layer `l`, create:
   - A static embedding matrix `M^l` of shape `[|V|, d_mem]`
   - A SwiGLU dynamic projection: `G^l(x) = SwiGLU(W1^l * x, W2^l * x)` where `W1, W2` have shape `[d_mem, d_embed]`
   - Learnable scalars `alpha^l`, `beta^l` (initialized to 1.0)
   - RMSNorm over the fused vector
   - A low-rank gate: `g = sigmoid(W_gate^l * h^l)` where `W_gate` has shape `[d_mem, d_model]` (low-rank factored)
   - An output projection: `y = RMSNorm(W_out^l * (e + g))` where `W_out` has shape `[d_model, d_mem]`

4. **Wire the module into the forward pass.** After computing `ffn_out = FFN(norm(h))`, also compute `meki_out = MemoryExpert(token_ids, h)`. Add both: `h = h + ffn_out + meki_out`. The memory expert branch runs in parallel with the FFN — no serial dependency.

5. **Train the model end-to-end.** Use the standard causal language modeling objective. The paper trains on 50B tokens with AdamW (lr schedule: cosine decay over 50k steps, beta1=0.9, beta2=0.95, gradient clip 1.0, BFloat16 mixed precision). The memory expert parameters train alongside the base model.

6. **Re-parameterize for inference.** After training, for each layer `l` and each token `t` in the vocabulary, pre-compute: `M_tilde^l[t] = alpha^l * RMSNorm(M^l[t] + beta^l * G^l(E_global[t]))`. Store the resulting `[|V|, d_mem]` table per layer. Discard `M^l`, `W1^l`, `W2^l`, `alpha^l`, `beta^l`, and the global embedding projection weights.

7. **Serialize the lookup tables to ROM format.** Write each layer's `M_tilde^l` as a flat float16 binary file. Organize as `layer_00.bin`, `layer_01.bin`, etc. These files are memory-mapped at runtime — no deserialization needed.

8. **Implement the inference-time module.** Replace the training memory expert with: (a) index into `M_tilde^l` by token ID to get `e^l`, (b) compute gate `g = sigmoid(W_gate * h)`, (c) project `y = RMSNorm(W_out * (e + g))`. Only `W_gate` and `W_out` (low-rank, small) remain in RAM.

9. **Enable asynchronous prefetching.** Because token IDs are known before the layer computes its hidden states, issue the ROM read for `M_tilde^l[token_id]` as soon as the token enters the layer — overlap the lookup with the attention and FFN computation. On UFS-4.0 storage (~4.2 GB/s), reading a 256-dim float16 vector (512 bytes) per token is negligible.

10. **Benchmark and validate.** Compare against the dense baseline on knowledge-intensive benchmarks (ARC-Challenge, HellaSwag, MMLU). Expect the largest gains on factual/knowledge tasks. Confirm inference latency is unchanged by profiling the end-to-end generation loop.

## Concrete Examples

**Example 1: Adding MeKi to a 1.5B Transformer for mobile deployment**

User: "I have a 1.5B parameter Transformer LLM that runs at 15 tok/s on a phone. I want better quality without slowing it down. Can you add MeKi-style memory experts?"

Approach:
1. Read the model config: 24 layers, d_model=2048, vocab=64000, d_embed=2048
2. Choose d_mem=256 (ROM cost: 24 * 64000 * 256 * 2 bytes = ~750 MB)
3. Add a `MemoryExpertLayer` module per Transformer block:

```python
class MemoryExpertLayer(nn.Module):
    def __init__(self, vocab_size, d_model, d_mem=256, gate_rank=64):
        super().__init__()
        # Training-time components
        self.static_embed = nn.Embedding(vocab_size, d_mem)
        self.dyn_w1 = nn.Linear(d_model, d_mem, bias=False)
        self.dyn_w2 = nn.Linear(d_model, d_mem, bias=False)
        self.alpha = nn.Parameter(torch.ones(1))
        self.beta = nn.Parameter(torch.ones(1))
        self.fuse_norm = RMSNorm(d_mem)
        # Shared (kept at inference)
        self.gate_proj = nn.Linear(d_model, d_mem, bias=False)
        self.out_proj = nn.Linear(d_mem, d_model, bias=False)
        self.out_norm = RMSNorm(d_mem)

    def forward(self, token_ids, hidden_states, global_embed):
        # Static path
        m_static = self.static_embed(token_ids)
        # Dynamic path (SwiGLU)
        e_global = global_embed(token_ids)
        m_dyn = F.silu(self.dyn_w1(e_global)) * self.dyn_w2(e_global)
        # Fuse
        expert_vec = self.alpha * self.fuse_norm(m_static + self.beta * m_dyn)
        # Gate (context-dependent)
        gate = torch.sigmoid(self.gate_proj(hidden_states))
        # Output
        return self.out_proj(self.out_norm(expert_vec + gate))
```

4. After training, run re-parameterization:

```python
def reparameterize(model):
    tables = {}
    global_embed = model.embed_tokens.weight  # [V, d_embed]
    for i, layer in enumerate(model.layers):
        me = layer.memory_expert
        all_ids = torch.arange(model.config.vocab_size)
        e_global = global_embed[all_ids]
        m_static = me.static_embed.weight
        m_dyn = F.silu(me.dyn_w1(e_global)) * me.dyn_w2(e_global)
        fused = me.alpha * me.fuse_norm(m_static + me.beta * m_dyn)
        tables[f"layer_{i:02d}"] = fused.half().cpu()
    return tables
```

5. Save each table as a flat binary for memory-mapped access at inference.

Output: Model quality improves by ~3-5 points on knowledge benchmarks, inference speed unchanged, ROM usage ~750 MB.

---

**Example 2: Replacing MoE routing with MeKi for latency-stable inference**

User: "Our MoE model has high latency variance on mobile because expert loading is unpredictable. Can we switch to MeKi-style deterministic lookup?"

Approach:
1. Identify the MoE layers — typically they replace the FFN every N layers
2. For each MoE layer, remove the router network and sparse expert matrices
3. Replace with a MeKi memory expert: one `[V, d_mem]` lookup table per layer, plus small gate and output projections in RAM
4. Retrain (or fine-tune) with the MeKi module wired in parallel with a single dense FFN
5. Re-parameterize and export lookup tables to ROM
6. Profile: expect latency variance to drop dramatically since every token follows the same code path (embedding lookup, not conditional routing)

Output: Latency p99 drops significantly because there are no routing-dependent memory access patterns. Throughput remains similar but quality may differ — validate on your task benchmarks.

---

**Example 3: Estimating ROM budget for a MeKi deployment**

User: "How much storage does MeKi add for a 3B model with 32 layers and 128k vocabulary?"

Approach:
1. Calculate: `storage = L * |V| * d_mem * bytes_per_param`
2. With d_mem=256, float16: `32 * 128000 * 256 * 2 = ~2.1 GB`
3. With d_mem=128 (reduced quality): `32 * 128000 * 128 * 2 = ~1.05 GB`
4. RAM overhead (gate + output projections per layer, low-rank): negligible (~5-10 MB total)
5. Recommend d_mem=256 if the device has 4+ GB free storage, d_mem=128 for tighter budgets

Output:
```
MeKi ROM Budget (3B model, 32 layers, 128k vocab, float16):
  d_mem=64:   ~524 MB   (minimal quality gain)
  d_mem=128:  ~1.05 GB  (moderate gain)
  d_mem=256:  ~2.1 GB   (recommended, best quality/storage ratio)
  d_mem=448:  ~3.67 GB  (diminishing returns)
RAM overhead: ~5-10 MB (gate + output projections only)
```

## Best Practices

- **Do:** Choose `d_mem` based on available device storage, not RAM. The lookup tables live in ROM/flash and impose no RAM pressure beyond the small gate and output projection matrices.
- **Do:** Run re-parameterization as a post-training step before deployment. Never ship the training-time dynamic projection weights — they waste storage and add unnecessary inference compute.
- **Do:** Use memory-mapped I/O (`mmap`) for the lookup tables at runtime. This lets the OS page in only the vectors actually accessed, and enables prefetching.
- **Do:** Prefetch the next layer's memory expert vector asynchronously while the current layer's attention and FFN are computing. Token IDs are known ahead of time.
- **Avoid:** Using MeKi with extremely large vocabularies (>500k tokens) unless storage is abundant — the tables scale linearly with `|V|`.
- **Avoid:** Applying MeKi to models where the FFN is already the bottleneck in RAM. MeKi adds a parallel branch; if the FFN's RAM weight movement is already saturating bandwidth, the gate/output projections (though small) add to the problem.

## Error Handling

- **RMSNorm numerical instability during re-parameterization:** When folding the dynamic projection, the fused vector may have outlier magnitudes. Compute RMSNorm in float32 before casting to float16 for storage.
- **Vocabulary mismatch:** If the tokenizer vocabulary changes after training, the lookup tables are invalidated. Always version-lock the tokenizer with the MeKi tables.
- **Storage I/O stalls:** On devices with slow storage (eMMC < 1 GB/s), the ROM lookup may introduce latency. Profile with `d_mem=64` first to verify I/O is not a bottleneck before scaling up.
- **Gate projection rank too high:** If `W_gate` is full-rank `[d_mem, d_model]`, it adds non-trivial RAM and compute. Use low-rank factorization (rank 32-64) to keep the gate lightweight.
- **Training divergence with large `d_mem`:** If the memory expert dominates the FFN early in training, the model may underfit. Initialize `alpha` and `beta` to small values (0.1) and let them grow, or use a warmup schedule for the memory expert learning rate.

## Limitations

- **Storage-bound, not compute-bound:** MeKi trades storage for quality. On devices with limited flash (< 2 GB free), the technique may not be viable for large vocabularies or many layers.
- **No benefit for compute-heavy tasks:** If the bottleneck is attention (long contexts), MeKi does not help — it augments the FFN path only.
- **Static knowledge:** The lookup tables are frozen after re-parameterization. Updating knowledge requires retraining and re-exporting. This is not suitable for rapidly changing factual knowledge.
- **Token-level, not sequence-level:** MeKi injects knowledge per-token, not per-sequence. It cannot learn sequence-level patterns that span multiple tokens in the expert vector itself (though the gate is context-aware).
- **Retraining required:** MeKi is not a post-hoc add-on. The memory expert modules must be trained jointly with the base model (or fine-tuned with substantial data). You cannot bolt MeKi onto a pretrained model without retraining.

## Reference

**Paper:** [MeKi: Memory-based Expert Knowledge Injection for Efficient LLM Scaling](https://arxiv.org/abs/2602.03359v1) — Ding, Liu, Kim, Hao, Lee (2026). Focus on Section 3 (Method) for the re-parameterization derivation and Section 4 (Experiments) for the edge-device latency profiling on Snapdragon 8 Elite.