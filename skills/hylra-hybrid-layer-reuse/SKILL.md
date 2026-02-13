---
name: "hylra-hybrid-layer-reuse"
description: "Implement HyLRA (Hybrid Layer Reuse Attention) for efficient long-context LLM inference. Profiles layer-wise sparsity, assigns full vs. sparse attention via dynamic programming, and reuses top-k token indices across tolerant layers. Use when: 'optimize long-context inference', 'reduce KV cache memory', 'implement sparse attention with layer reuse', 'speed up attention for long sequences', 'profile layer sensitivity for attention sparsity', 'implement HyLRA from the paper'."
---

# HyLRA: Hybrid Layer Reuse Attention for Efficient Long-Context Inference

This skill enables Claude to implement the HyLRA framework, which accelerates long-context LLM inference by classifying transformer layers as either *sensitive* (requiring full attention) or *tolerant* (safe to reuse top-k token indices from a preceding layer). The core insight is that consecutive layers often attend to nearly identical critical tokens, so tolerant layers can skip the expensive quadratic attention score computation entirely and operate on a sparse subset of key-value pairs inherited from the last full-attention layer. An offline dynamic programming pass determines the optimal assignment, yielding 6--46% throughput gains with less than 1% accuracy loss.

## When to Use

- When the user asks to **reduce inference latency for long-context LLMs** (16K+ token sequences) by exploiting attention sparsity patterns.
- When implementing a **layer-wise attention policy** that decides which transformer layers need full attention and which can reuse sparse indices.
- When the user wants to **profile layer sensitivity** to understand which layers tolerate approximation and which do not.
- When building an **index-guided KV cache offloading** system that prefetches only critical KV blocks from host memory.
- When the user wants to **compare or integrate sparse attention methods** (e.g., QUEST, StreamingLLM, SnapKV) with a hybrid reuse strategy.
- When optimizing a **serving pipeline** where attention is the throughput bottleneck at long sequence lengths.

## Key Technique

**Dual Characteristic of Attention Mechanics.** HyLRA rests on two empirical observations. First, *intra-layer sensitivity*: certain layers exhibit large output distortion (measured by RNMSE and KL divergence) when their attention is approximated with top-k sparsity, meaning they must retain full dense attention. Second, *inter-layer similarity*: consecutive layers share a high overlap of critical token indices (measured as `|S_i ∩ S_j| / k` for top-k sets `S_i`, `S_j`), meaning a tolerant layer can inherit its predecessor's token selection without recomputing attention scores.

**Dynamic Programming Policy Optimization.** Layer assignment is formulated as a shortest-path problem on a similarity matrix. Each layer is a node; a *vertical move* from layer `i` to layer `j+1` means "reuse indices from layer `i`" (allowed only if overlap exceeds threshold `θ`); a *reset jump* means "run full attention to refresh indices." The DP minimizes the count of full-attention layers (primary objective) and maximizes cumulative overlap similarity (secondary tiebreaker). The resulting policy `π` maps each layer to either `FULL` or `REUSE(source_layer)`.

**Inference Execution.** At runtime, full-attention layers compute standard scaled dot-product attention and extract top-k indices. Reuse-attention layers inherit the index set from their designated source layer, gather only the corresponding K/V entries, compute attention on this sparse subset, and produce output — bypassing the `O(n²)` score computation entirely. For KV caches that exceed device memory, indices from the preceding full-attention layer guide selective prefetching of blocks from host RAM, cutting PCIe transfer volume.

## Step-by-Step Workflow

1. **Select calibration data.** Gather a small representative dataset (a few hundred sequences at the target context length) that mirrors the deployment distribution. This dataset drives the offline profiling stage.

2. **Profile intra-layer sensitivity.** For each layer `l` in the model, replace its attention with top-k sparse attention (e.g., k = 2048) while keeping all other layers at full attention. Measure output distortion via RNMSE between the sparse and full output features, and KL divergence of the next-token distribution. Record `sensitivity[l]`.

3. **Profile inter-layer similarity.** For every pair of layers `(i, j)`, compute the top-k index overlap: `overlap[i][j] = |topk_indices_i ∩ topk_indices_j| / k`. Store the resulting `L × L` similarity matrix. Expect high values along the diagonal (consecutive layers) and sparse off-diagonal structure.

4. **Set the similarity threshold `θ`.** Choose a threshold (e.g., `θ = 0.85`) below which reuse is disallowed. This is the quality knob: higher `θ` preserves accuracy but allows fewer reuse layers; lower `θ` increases speedup at the cost of fidelity.

5. **Run dynamic programming to derive the layer policy.** Initialize DP state at layer 0 as `FULL`. For each subsequent layer `j`, evaluate two transitions from every reachable state `(source, j-1)`: (a) vertical move — mark layer `j` as `REUSE(source)` if `overlap[source][j] ≥ θ`; (b) reset jump — mark layer `j` as `FULL`. Retain only Pareto-optimal paths (minimum full-attention count, maximum cumulative similarity). Extract the final policy `π[l]` for each layer.

6. **Implement the hybrid attention kernel.** Write two code paths in the attention module: a standard full-attention path that computes scores and extracts top-k indices, and a reuse path that accepts inherited indices, gathers sparse K/V slices, and computes attention on the reduced set. Use block-level granularity (e.g., 128-token blocks) for coalesced GPU memory access.

7. **Integrate the policy into the model forward pass.** Before inference, load the policy `π`. In each layer's forward method, branch on `π[l]`: if `FULL`, execute dense attention and store top-k indices; if `REUSE`, fetch indices from the source layer and execute sparse attention.

8. **Implement index-guided KV cache offloading (optional).** When KV caches exceed device HBM, keep full caches on the host. Before a reuse layer executes, use the inherited top-k block indices to selectively prefetch only the needed KV blocks over PCIe, avoiding full cache transfers.

9. **Validate on downstream benchmarks.** Run the model on long-context benchmarks (e.g., LongBench v2, RULER, Needle-in-a-Haystack) and verify accuracy degradation stays below 1%. Measure end-to-end throughput at multiple sequence lengths (8K, 16K, 32K, 60K+).

10. **Tune and iterate.** Adjust `θ`, `k`, and block size based on results. Lower `θ` if throughput is insufficient; raise it if accuracy drops. Re-run the DP to get the updated policy — it executes in milliseconds since it is offline.

## Concrete Examples

**Example 1: Profiling layer sensitivity for a 32B model**

User: "I want to figure out which layers in my Qwen3-32B model are sensitive to sparse attention."

Approach:
1. Load the model and calibration dataset (e.g., 200 sequences of 32K tokens from the target domain).
2. For each of the 64 layers, swap in top-k=2048 sparse attention for that single layer while keeping all others dense.
3. Compute RNMSE of the layer's output features and KL divergence of the final logits vs. the full-attention baseline.
4. Rank layers by sensitivity. Expect early layers (0-3) and a few mid-network layers to show high RNMSE (>0.1), while most layers show low values (<0.02).

Output:
```
Layer  RNMSE   KL-Div   Classification
  0    0.251   0.034    SENSITIVE
  1    0.187   0.021    SENSITIVE
  2    0.016   0.002    TOLERANT
  3    0.012   0.001    TOLERANT
  ...
 31    0.243   0.029    SENSITIVE  (mid-network critical layer)
  ...
 63    0.009   0.001    TOLERANT

Sensitive layers: [0, 1, 31, 47]  (4/64 = 6.25%)
Tolerant layers:  60/64 = 93.75%
```

**Example 2: Running the DP to derive a layer policy**

User: "Given the overlap matrix, compute the optimal HyLRA policy with θ=0.85."

Approach:
1. Parse the `L × L` overlap matrix from the profiling stage.
2. Initialize DP: `dp[0] = (cost=1, sim=0, source=0, type=FULL)`.
3. For each layer `j` from 1 to L-1, evaluate vertical moves and reset jumps from all reachable states.
4. Prune dominated states (higher cost and lower similarity).
5. Backtrack to extract the policy.

Output:
```python
# Resulting policy for a 64-layer model (θ=0.85)
policy = {
    0:  ("FULL", None),     # Must compute fresh indices
    1:  ("REUSE", 0),       # Overlap with layer 0 = 0.91
    2:  ("REUSE", 0),       # Overlap with layer 0 = 0.87
    3:  ("REUSE", 0),       # Overlap with layer 0 = 0.86
    4:  ("FULL", None),     # Overlap dropped below θ
    5:  ("REUSE", 4),       # Overlap with layer 4 = 0.93
    ...
    31: ("FULL", None),     # Sensitive layer — forced FULL
    32: ("REUSE", 31),      # Overlap with layer 31 = 0.89
    ...
}
# Full attention layers: 12/64 (18.75%)
# Reuse layers:          52/64 (81.25%)
# Estimated speedup at 32K tokens: ~1.30x
```

**Example 3: Implementing the hybrid forward pass**

User: "Show me how to modify a transformer forward pass to support HyLRA."

Approach:
1. Store the policy and a shared index buffer accessible across layers.
2. In each layer's attention, branch on the policy.
3. For FULL layers, compute dense attention and write top-k indices to the buffer.
4. For REUSE layers, read indices from the buffer and compute sparse attention.

Output:
```python
class HyLRAAttention(nn.Module):
    def __init__(self, base_attn, layer_id, policy, k=2048, block_size=128):
        super().__init__()
        self.base_attn = base_attn
        self.layer_id = layer_id
        self.policy = policy          # dict: layer_id -> ("FULL"|"REUSE", source_id)
        self.k = k
        self.block_size = block_size
        # Shared across layers via external registry
        self.index_buffer = {}        # layer_id -> top-k block indices

    def forward(self, Q, K, V, **kwargs):
        mode, source = self.policy[self.layer_id]

        if mode == "FULL":
            # Standard attention: O(n^2) score computation
            scores = torch.matmul(Q, K.transpose(-1, -2)) / math.sqrt(Q.size(-1))
            attn_weights = F.softmax(scores, dim=-1)
            output = torch.matmul(attn_weights, V)

            # Extract and store top-k block indices for downstream reuse
            block_scores = scores.unfold(-1, self.block_size, self.block_size).mean(-1)
            topk_blocks = block_scores.topk(self.k // self.block_size, dim=-1).indices
            self.index_buffer[self.layer_id] = topk_blocks
            return output

        else:  # REUSE
            # Inherit indices from source layer — no score computation
            topk_blocks = self.index_buffer[source]

            # Gather sparse K, V using block indices
            K_sparse = gather_blocks(K, topk_blocks, self.block_size)
            V_sparse = gather_blocks(V, topk_blocks, self.block_size)

            # Sparse attention: O(n * k) instead of O(n^2)
            scores = torch.matmul(Q, K_sparse.transpose(-1, -2)) / math.sqrt(Q.size(-1))
            attn_weights = F.softmax(scores, dim=-1)
            output = torch.matmul(attn_weights, V_sparse)
            return output
```

## Best Practices

- **Do:** Profile on data that matches your deployment distribution. Sensitivity patterns shift across domains (code vs. natural language vs. math reasoning).
- **Do:** Use block-level granularity (64--128 tokens per block) rather than individual token indices. This enables coalesced GPU memory access and avoids scatter/gather overhead.
- **Do:** Always force the first 1--2 layers and the final layer to FULL attention. These are empirically the most sensitive across all tested architectures.
- **Do:** Re-run profiling and the DP when switching model architectures or quantization schemes (e.g., FP16 vs. W8A8). Sensitivity profiles change with precision.
- **Avoid:** Setting `θ` below 0.80. Overlap below this level introduces measurable accuracy degradation, especially on reasoning-heavy tasks.
- **Avoid:** Applying HyLRA to very short sequences (<8K tokens). The overhead of index management exceeds savings when the attention matrix is already small. HyLRA's gains scale with sequence length.

## Error Handling

| Problem | Symptom | Fix |
|---------|---------|-----|
| Accuracy drops >1% after applying policy | Downstream benchmark regression | Raise `θ` by 0.05 increments; force additional sensitive layers to FULL |
| No speedup observed | Throughput unchanged or worse | Check that reuse layers actually skip score computation — a common bug is computing scores then discarding them. Verify sequence length is long enough (>16K). |
| Index buffer stale after KV cache rotation | Garbage indices in reuse layers during streaming/generation | Clear the index buffer whenever the KV cache is evicted or rotated; re-derive indices at the next FULL layer. |
| OOM during profiling | GPU memory exhaustion with long calibration sequences | Profile with gradient checkpointing enabled or use activation offloading. Only one layer's sparse output needs to be in memory at a time. |
| Overlap matrix has no entries above `θ` for a layer span | DP forces every layer to FULL, eliminating all reuse | The model may have low inter-layer similarity in that region. Lower `θ` locally or accept those layers as sensitive. |

## Limitations

- **Offline profiling required.** The DP policy is derived from calibration data before deployment. It does not adapt online to novel input distributions that shift layer sensitivity at runtime.
- **Fixed top-k budget.** The same `k` applies globally; some layers may benefit from larger or smaller budgets. Per-layer `k` tuning is possible but increases the profiling search space.
- **Architecture-specific.** Policies must be re-derived for each model architecture (and each quantization configuration). A policy for DeepSeek-R1 does not transfer to Llama or Qwen.
- **Prefill vs. decode asymmetry.** HyLRA primarily accelerates the prefill phase (processing the full prompt) where attention is quadratic. During autoregressive decoding (single-token steps), the quadratic bottleneck is less pronounced and gains are smaller.
- **Not composable with all KV compression methods.** Techniques that fundamentally alter which tokens are stored in the KV cache (e.g., heavy eviction policies) may conflict with the assumption that top-k indices remain valid across layers.

## Reference

**Paper:** [HyLRA: Hybrid Layer Reuse Attention for Efficient Long-Context Inference](https://arxiv.org/abs/2602.00777v1) (Ai et al., 2026). Focus on Section 3 (sparsity profiling methodology), Algorithm 1 (DP policy optimization), and Algorithm 2 (hybrid inference execution) for implementation details.