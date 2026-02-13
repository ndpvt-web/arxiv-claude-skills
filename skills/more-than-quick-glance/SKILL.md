---
name: "more-than-quick-glance"
description: "Implement LASER-KV-style KV-cache compression for LLM inference pipelines using block-wise accumulative budgeting and hybrid exact-attention/LSH token selection. Use when: 'optimize KV cache for long context', 'compress KV cache without losing accuracy', 'implement LASER-KV', 'reduce LLM memory for 128k context', 'block-wise cache eviction strategy', 'fix attention-score greedy bias in cache pruning'."
---

# LASER-KV: Block-Wise Accumulative KV-Cache Compression with Hybrid Token Selection

This skill enables Claude to implement KV-cache compression systems based on the LASER-KV framework (Layer Accumulated Selection with Exact-LSH Recall). Instead of relying solely on attention scores to decide which tokens to keep — a greedy approach that degrades 15-30% on long-context tasks — this technique partitions the cache budget into syntactic anchors, exact-attention winners, and LSH-recalled structural tokens, processed block-by-block with accumulative budgets. The result is stable performance up to 128k context with 75% cache reduction.

## When to Use

- When the user needs to implement or optimize a KV-cache eviction/compression strategy for long-context LLM serving
- When deploying models at 16k-128k+ context lengths and hitting GPU memory limits from KV-cache growth
- When existing cache pruning (H2O, SnapKV, PyramidKV, FINCH) shows degradation on recall-heavy tasks like multi-hop QA
- When the user asks to reduce inference memory without retraining or changing model architecture
- When building a token selection policy that goes beyond raw attention scores
- When implementing block-wise processing pipelines for streaming or chunked LLM inference

## Key Technique

**The Greedy Bias Problem.** Standard KV-cache compression ranks tokens by attention scores and evicts the lowest-scoring ones. This is greedy: it preserves tokens the model is *currently* attending to but discards "supporting tokens" that are structurally relevant but not actively queried. On benchmarks requiring recall of earlier context (Babilong QA tasks), this causes 15-30% accuracy drops at high compression ratios.

**Block-Wise Accumulative Budgeting.** LASER-KV divides the input context into fixed-size blocks (e.g., 4096 tokens) and processes each independently. The total budget per block is computed as `B = floor(2 * r * S_block * L / (S_block + L))` where `r` is the compression ratio (0.25 = keep 25%), `S_block` is block size, and `L` is total context length. A protection divisor `n` (default: 4) splits this budget into three pools: `B/n` for global anchors (initial tokens), `B/n` for a local sliding window (recent tokens), and the remainder `B - 2B/n` for the long-term recall pool. This accumulative approach avoids the recursive degradation seen in fixed-summary-size methods.

**Hybrid Exact-Attention + LSH Selection.** Within the long-term recall pool, token selection uses a two-phase hybrid. First, attention scores are summed across *all* layers and heads (global consensus), and the top `alpha * B_long` tokens are kept (default alpha=0.75). Then, for the remaining candidates, Locality-Sensitive Hashing computes hash collision probabilities against the query, and the top `(1-alpha) * B_long` tokens are selected. This LSH phase recovers structurally important tokens that no single attention head strongly attends to but that share geometric similarity with the query in embedding space.

## Step-by-Step Workflow

1. **Compute the per-block budget.** Given total context length `L`, block size `S_block` (4096), and compression ratio `r` (0.25), calculate: `B = floor(2 * r * S_block * L / (S_block + L))`. This is the total number of tokens retained per block.

2. **Partition the budget with protection divisor `n=4`.** Allocate `B/n` slots for global anchors (first tokens of the sequence), `B/n` slots for the local sliding window (most recent tokens), and `B_long = B - 2*B/n` slots for the long-term memory pool.

3. **Divide the input into non-overlapping blocks of `S_block` tokens.** Process blocks sequentially. For each block, expand the scoring window to include the tail of the previous block (look-back) to avoid boundary artifacts where tokens at block edges lack sufficient context.

4. **Compute global attention consensus scores.** For each candidate token in the current block, sum its attention scores across all layers and all attention heads: `S_exact = sum over (layers, heads) of Attention(Q, C)`. This captures tokens important to any head, even specialized ones like induction heads.

5. **Select the exact-attention winners.** Take the top `alpha * B_long` tokens by `S_exact` score (default alpha=0.75). These are the highest-value tokens by direct attention measurement.

6. **Apply LSH recall on the residual candidates.** For tokens not selected in step 5, compute LSH collision scores: `S_lsh = (1/R) * sum over hash functions of indicator[h_r(Q) == h_r(k)]`. Select the top `(1-alpha) * B_long` tokens. This recovers structurally relevant tokens missed by greedy attention selection.

7. **Merge all selected token sets.** The final retained set for the block is: global anchors + local window tokens + exact-attention winners + LSH-recalled tokens. Append these to the compressed KV cache.

8. **Repeat for all blocks, accumulating into the output cache.** Each block's selections are additive (append-only), avoiding recursive recompression that causes compounding errors.

9. **Forward the compressed KV cache to the decoding phase.** The compressed cache is a drop-in replacement for the full cache during autoregressive generation. No architectural changes are needed.

10. **Tune hyperparameters for the deployment target.** Adjust `r` (compression aggressiveness), `n` (syntactic protection), `alpha` (exact vs. LSH balance), and `S_block` (granularity) based on task type and memory constraints.

## Concrete Examples

**Example 1: Implementing a KV-Cache Compressor for a Llama-3 Serving Pipeline**

User: "I'm serving Llama-3.1-8B at 64k context and running out of GPU memory. Implement a KV-cache compression module that keeps 25% of tokens without destroying recall accuracy."

Approach:
1. Create a `LaserKVCompressor` class that takes the full KV cache tensors after prefill
2. Implement the budget formula: for L=65536, S_block=4096, r=0.25 => B = floor(2*0.25*4096*65536 / (4096+65536)) = floor(1524.5) = 1524 tokens per block
3. Partition: n=4 gives 381 anchor slots, 381 window slots, 762 long-term slots
4. For each of the 16 blocks: sum attention across all 32 heads and 32 layers, select top-571 by score (alpha=0.75 of 762), then LSH-select top-191 from residuals
5. Concatenate all block outputs into the compressed cache

Output:
```python
import torch
import math

class LaserKVCompressor:
    def __init__(self, compression_ratio=0.25, block_size=4096,
                 protection_divisor=4, alpha=0.75, num_hash_rounds=32):
        self.r = compression_ratio
        self.s_block = block_size
        self.n = protection_divisor
        self.alpha = alpha
        self.R = num_hash_rounds

    def compute_budget(self, total_length: int) -> int:
        return math.floor(
            2 * self.r * self.s_block * total_length
            / (self.s_block + total_length)
        )

    def partition_budget(self, B: int):
        anchor_slots = B // self.n
        window_slots = B // self.n
        long_term_slots = B - anchor_slots - window_slots
        return anchor_slots, window_slots, long_term_slots

    def lsh_scores(self, queries, candidates, dim):
        """Compute LSH collision probability scores."""
        # Random hyperplane LSH
        planes = torch.randn(self.R, dim, device=queries.device)
        q_hashes = (queries @ planes.T) > 0        # [num_q, R]
        c_hashes = (candidates @ planes.T) > 0      # [num_c, R]
        # Average collision rate across hash rounds
        collisions = (q_hashes.unsqueeze(1) == c_hashes.unsqueeze(0)).float()
        return collisions.mean(dim=(0, 2))  # [num_c]

    def compress(self, kv_cache, attention_weights):
        """
        kv_cache: tuple of (K, V) each [batch, heads, seq_len, dim]
        attention_weights: [batch, heads, layers, seq_len, seq_len]
        Returns: compressed (K, V) with reduced seq_len
        """
        seq_len = kv_cache[0].shape[2]
        B = self.compute_budget(seq_len)
        anchor_n, window_n, long_n = self.partition_budget(B)
        num_blocks = math.ceil(seq_len / self.s_block)

        selected_indices = set(range(anchor_n))  # global anchors

        for block_idx in range(num_blocks):
            start = block_idx * self.s_block
            end = min(start + self.s_block, seq_len)
            block_indices = list(range(start, end))

            # Sum attention across all layers and heads for global consensus
            attn_scores = attention_weights[:, :, :, -1, start:end]
            global_scores = attn_scores.sum(dim=(0, 1, 2))  # [block_len]

            # Exact attention selection
            exact_k = int(self.alpha * long_n)
            _, top_exact = torch.topk(global_scores, min(exact_k, len(global_scores)))
            exact_selected = {block_indices[i] for i in top_exact.tolist()}

            # LSH on residuals
            residual_mask = torch.ones(len(block_indices), dtype=torch.bool)
            residual_mask[top_exact] = False
            residual_indices = [block_indices[i] for i, m in enumerate(residual_mask) if m]

            if residual_indices and (long_n - exact_k) > 0:
                K = kv_cache[0]
                residual_keys = K[0, 0, residual_indices, :]
                query = K[0, 0, -1:, :]
                lsh_sc = self.lsh_scores(query, residual_keys, K.shape[-1])
                lsh_k = min(long_n - exact_k, len(residual_indices))
                _, top_lsh = torch.topk(lsh_sc, lsh_k)
                lsh_selected = {residual_indices[i] for i in top_lsh.tolist()}
            else:
                lsh_selected = set()

            selected_indices.update(exact_selected)
            selected_indices.update(lsh_selected)

        # Add local window
        window_indices = set(range(max(0, seq_len - window_n), seq_len))
        selected_indices.update(window_indices)

        idx = sorted(selected_indices)
        idx_tensor = torch.tensor(idx, device=kv_cache[0].device)
        compressed_k = kv_cache[0][:, :, idx_tensor, :]
        compressed_v = kv_cache[1][:, :, idx_tensor, :]
        return (compressed_k, compressed_v)
```

**Example 2: Adding LSH Recall to an Existing Attention-Based Eviction Policy**

User: "I already have a Top-K attention-based cache eviction. Accuracy tanks on long documents. How do I add the LSH recall component from LASER-KV?"

Approach:
1. After the existing Top-K selection, collect the *residual* (unchosen) candidates
2. Implement random-hyperplane LSH hashing for the residual keys against the current query
3. Score residuals by average hash collision rate across R rounds
4. Select additional `(1-alpha) * budget` tokens from residuals by LSH score
5. Merge with the existing Top-K set

Output:
```python
def add_lsh_recall(query, candidate_keys, existing_topk_indices,
                   total_budget, alpha=0.75, num_rounds=32):
    """
    Augment a greedy Top-K selection with LSH recall.

    query:                [1, dim] - current query vector
    candidate_keys:       [seq_len, dim] - all candidate key vectors
    existing_topk_indices: list of int - indices already selected by attention
    total_budget:         int - total number of tokens to retain
    """
    all_indices = set(range(candidate_keys.shape[0]))
    residual_indices = sorted(all_indices - set(existing_topk_indices))

    if not residual_indices:
        return existing_topk_indices

    lsh_budget = int((1 - alpha) * total_budget)
    residual_keys = candidate_keys[residual_indices]

    # Random hyperplane LSH
    dim = query.shape[-1]
    planes = torch.randn(num_rounds, dim, device=query.device)
    q_hash = (query @ planes.T) > 0           # [1, R]
    r_hash = (residual_keys @ planes.T) > 0   # [num_residual, R]

    collision_rate = (q_hash == r_hash).float().mean(dim=1)  # [num_residual]
    k = min(lsh_budget, len(residual_indices))
    _, top_lsh = torch.topk(collision_rate, k)

    lsh_selected = [residual_indices[i] for i in top_lsh.tolist()]
    return sorted(set(existing_topk_indices) | set(lsh_selected))
```

**Example 3: Configuring Block Size and Protection Divisor for a Specific Deployment**

User: "I'm deploying at 128k context with tight memory. What parameters should I use for LASER-KV compression?"

Approach:
1. Calculate budget at different compression ratios
2. Show the effect of protection divisor on syntactic vs recall allocation
3. Recommend block size based on context length

Output:
```
For L=131072, S_block=4096, the budget formula gives:

  r=0.25: B = floor(2*0.25*4096*131072 / (4096+131072)) = 1993 tokens/block
  r=0.10: B = floor(2*0.10*4096*131072 / (4096+131072)) =  797 tokens/block

With protection divisor n=4 at r=0.25:
  - Global anchors:    498 tokens (initial context preserved)
  - Local window:      498 tokens (recent generation coherence)
  - Long-term recall:  997 tokens (split 75/25 exact/LSH = 747 + 250)

Recommendations for 128k:
  - Use S_block=4096 (matches paper's tested configuration)
  - Start with n=4; increase to n=6 if generation fluency is fine but recall drops
  - Keep alpha=0.75 unless your task is heavily recall-dependent (then try 0.6)
  - Total blocks: 32 at 128k — each processed independently
  - Expected memory: ~25% of full KV cache + O(seq_len * R) for LSH tables
```

## Best Practices

- **Do:** Sum attention scores across *all* layers and heads before ranking tokens. Single-head or single-layer scores miss tokens important to specialized heads (induction heads, positional heads).
- **Do:** Use the look-back mechanism at block boundaries — include the tail of the previous block in the scoring window to prevent edge tokens from being unfairly evicted due to insufficient context.
- **Do:** Keep the append-only accumulation pattern. Never recompress already-compressed blocks; compounding errors degrade recall rapidly.
- **Do:** Profile the LSH overhead separately. The hash computation is `O(|C| * R * d_h)` — with large candidate sets and many hash rounds, this can become the bottleneck.
- **Avoid:** Setting `alpha=1.0` (pure attention-based selection). This reverts to the greedy bias the paper identifies as the core failure mode.
- **Avoid:** Using very small block sizes (< 1024 tokens). Small blocks give insufficient local context for accurate attention scoring, increasing the chance of evicting important tokens.

## Error Handling

- **Budget exceeds block size:** If `B >= S_block`, no compression is needed for that block — retain all tokens. This happens at low total context lengths where compression is unnecessary.
- **Empty residual set:** If the exact-attention phase selects all candidates (small block, large alpha), the LSH phase has nothing to select. This is fine — skip LSH gracefully.
- **Degenerate attention scores:** If attention is uniform across all tokens (e.g., early layers in some models), the exact phase provides little signal. Consider increasing `(1-alpha)` to rely more on LSH in these layers, or skip compression for uniform-attention layers.
- **Hash collision degeneracy:** With very low-dimensional head spaces, LSH collisions become noisy. Increase `R` (hash rounds) or use multi-probe LSH to compensate.

## Limitations

- **Prefill-phase only.** LASER-KV operates during the prefill stage. It does not handle dynamic eviction during autoregressive decoding — a separate strategy is needed if the cache grows during generation.
- **Compute overhead.** The global attention consensus (summing across all layers/heads) and LSH scoring add latency to the prefill step. For latency-sensitive applications with short contexts (< 8k), the overhead may outweigh the memory savings.
- **Task-dependent tuning.** The default alpha=0.75 works well on Babilong QA tasks, but summarization or creative generation tasks may need different balances. There is no universal configuration.
- **Single-model validation.** The paper validates on Llama-3 variants (8B). Behavior on architectures with different attention patterns (e.g., MoE models, GQA with fewer KV heads) is unverified.
- **No learned components.** All selection is heuristic (attention scores + LSH). A learned selection policy could potentially outperform, but at the cost of requiring training data and fine-tuning.

## Reference

**Paper:** [More Than a Quick Glance: Overcoming the Greedy Bias in KV-Cache Compression](https://arxiv.org/abs/2602.02199v1) — Sood, Sharma, Agrawal (2026). Look for: Algorithm 1 (Exact-LSH Selection Policy), the accumulative budget formula in Section 3, and the Babilong ablation tables showing the 15-30% degradation of attention-only methods.