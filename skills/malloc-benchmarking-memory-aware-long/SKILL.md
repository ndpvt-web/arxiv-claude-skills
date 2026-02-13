---
name: "malloc-benchmarking-memory-aware-long"
description: "Apply memory-aware compression strategies to long-sequence recommendation systems. Benchmark KV-cache compression techniques (eviction, merging, head-sharing, quantization) adapted from LLMs for sequential recommendation models like HSTU. Triggers: 'compress recommendation sequences', 'reduce KV cache memory in recommender', 'benchmark memory compression for recommendations', 'optimize long user history inference', 'MALLOC recommendation benchmark', 'memory-efficient sequential recommendation'"
---

# MALLOC: Memory-Aware Long Sequence Compression for Sequential Recommendation

This skill enables Claude to apply and benchmark memory compression strategies for large-scale sequential recommendation systems. Drawing from the MALLOC framework, it covers five granularities of KV-cache compression — sequence-level, token-level, head-level, precision-level, and architecture-level — originally developed for LLMs but systematically evaluated for recommendation workloads where billions of users each generate thousands of interactions. The core insight is that techniques like multi-query attention (MQA), grouped-query attention (GQA), and multi-head latent attention (MLA) form a Pareto frontier, delivering near-native accuracy while cutting compute from ~4.36G MACs to ~276M MACs, whereas aggressive token pruning (H2O, SnapKV) achieves the smallest memory footprint (~2 MB vs. 128 MB native) at a steeper accuracy cost.

## When to Use

- When the user needs to reduce GPU memory consumption for a transformer-based sequential recommender that caches intermediate states per user
- When building a production recommendation system serving billions of users and needing to store KV caches efficiently
- When the user asks how to adapt LLM KV-cache compression techniques (H2O, SnapKV, Linformer, GQA, KIVI) for recommendation tasks
- When evaluating tradeoffs between recommendation accuracy (GAUC, NDCG, Recall) and inference memory/latency
- When implementing HSTU or similar hierarchical transformer recommenders with long user interaction sequences (512-1024+ items)
- When the user wants to benchmark multiple compression strategies systematically against each other on recommendation datasets

## Key Technique

**The Memory-Latency Dilemma in Recommendation.** Modern sequential recommenders like HSTU pre-store intermediate KV-cache states for each user to avoid quadratic recomputation on every request. At scale (e.g., batch size 1000 with 1024-length sequences), native caching demands ~545 GB of memory. Without caching, computation surges by ~450x. MALLOC frames this as a structured optimization problem: compress the cached states while preserving recommendation quality.

**Five-Granularity Compression Taxonomy.** MALLOC classifies 16 compression methods across five levels: (1) **Sequence-level** methods like Linformer project keys/values to lower dimensions via fixed linear maps, and Reformer uses locality-sensitive hashing to limit attention scope. (2) **Token-level** methods either merge tokens (Longformer's sliding-window + global tokens, Activation Beacon's learnable aggregation tokens) or prune them (H2O retains high-impact tokens, SnapKV keeps persistently-attended tokens). (3) **Head-level** methods reduce redundancy across attention heads — MQA shares a single KV pair across all heads, GQA groups heads to share KV pairs, and MLA applies low-rank joint compression. (4) **Precision-level** methods like KIVI apply 2-bit quantization to cached states, while IntactKV preserves full precision for critical tokens only. (5) **Architecture-level** alternatives like RWKV replace attention entirely with gated RNN mechanisms.

**Key Finding: Head-Level Methods Dominate the Pareto Frontier.** Across three datasets (Amazon-Electronic, MicroVideo1.7M, KuaiVideo), head-level compression (MQA, GQA, MLA) consistently delivered accuracy comparable to native full caching while reducing MACs by ~16x and memory by ~8x. Token-pruning methods (H2O, SnapKV) achieved the most extreme memory savings (~64x reduction to 2 MB) but with measurable accuracy degradation. Precision-level quantization offered a middle ground. The optimal strategy depends on the specific memory budget and accuracy tolerance.

## Step-by-Step Workflow

1. **Profile the current memory bottleneck.** Measure peak KV-cache memory per inference request using `torch.cuda.max_memory_allocated()` or the `ptflops` package. Record baseline MACs (dense layers + attention projections), memory in MB, and latency in ms at your target batch size.

2. **Classify the compression budget.** Determine whether the constraint is primarily memory (storage cost per user), latency (inference speed), or compute (MACs). This determines which granularity of compression to prioritize:
   - Memory-constrained: Token pruning (H2O, SnapKV) or precision (KIVI)
   - Compute-constrained: Head-level (MQA, GQA, MLA)
   - Balanced: Head-level methods, which form the Pareto frontier

3. **Select candidate compression methods.** Choose 3-4 methods spanning different granularities for benchmarking. A strong default set: MLA (head-level), SnapKV (token-level), KIVI (precision-level), and Longformer (token-merging).

4. **Implement KV-cache compression in the attention layer.** Modify the model's attention mechanism to intercept keys and values before caching. For head-level methods, restructure the projection layers. For token-level methods, add a selection/pruning step after self-attention computes importance scores. For precision-level, wrap the cache write/read with quantize/dequantize ops.

5. **Configure compression ratios relative to sequence length.** For a 1024-token sequence, test retention ratios of 25%, 50%, and 75%. For head-level methods, test group sizes of 2, 4, and 8 heads per shared KV pair. For quantization, test 8-bit, 4-bit, and 2-bit.

6. **Run evaluation on recommendation metrics.** Compute GAUC (grouped AUC), NDCG@k, and Recall@k on held-out test data. Use the standard 70-10-20 train-validation-test split. Report metrics alongside memory consumption and MACs.

7. **Plot the Pareto frontier.** Create a scatter plot with accuracy (GAUC) on the y-axis and memory (MB) or MACs on the x-axis. Identify non-dominated solutions — methods where no other method is simultaneously better on both axes.

8. **Validate on tail users.** Test compression specifically on users with the longest interaction histories (top 10th percentile by sequence length) since compression effects are most pronounced there. Compare per-user accuracy degradation distributions.

9. **Implement the selected method with adaptive thresholds.** Deploy the Pareto-optimal method with runtime memory monitoring. If memory exceeds the budget, increase compression aggressively (e.g., drop from 50% to 25% retention). If memory is well within budget, relax compression to recover accuracy.

10. **Benchmark end-to-end serving throughput.** Measure requests-per-second at the target batch size with and without compression. Report the full resource profile: peak GPU memory, average latency (p50, p99), and accuracy delta vs. native caching.

## Concrete Examples

**Example 1: Adding GQA compression to an HSTU recommender**

User: "My HSTU-based recommender uses 8 attention heads and caches KV states for 1024-length user sequences. Memory per request is 128 MB. I need to get it under 20 MB."

Approach:
1. The 128 MB baseline comes from 8 heads x 2 (K+V) x 1024 tokens x 256 dim x 4 bytes (fp32).
2. GQA with 2 groups (4 heads sharing each KV pair) cuts memory by 4x to ~32 MB.
3. MLA with low-rank factor 4 cuts further to ~16 MB.
4. Recommend MLA since it meets the 20 MB target.

Implementation:
```python
import torch
import torch.nn as nn

class MLAAttention(nn.Module):
    """Multi-head Latent Attention for KV-cache compression in HSTU."""
    def __init__(self, d_model=256, n_heads=8, rank=64):
        super().__init__()
        self.n_heads = n_heads
        self.head_dim = d_model // n_heads
        self.rank = rank  # Low-rank compression dimension

        # Queries remain full-rank
        self.W_q = nn.Linear(d_model, d_model)
        # Keys and values projected through low-rank bottleneck
        self.W_kv_down = nn.Linear(d_model, rank)  # Compress
        self.W_k_up = nn.Linear(rank, d_model)      # Expand for keys
        self.W_v_up = nn.Linear(rank, d_model)      # Expand for values
        self.W_o = nn.Linear(d_model, d_model)

    def forward(self, x, kv_cache=None):
        B, T, D = x.shape
        q = self.W_q(x).view(B, T, self.n_heads, self.head_dim).transpose(1, 2)

        # Compress KV to low-rank representation for caching
        kv_compressed = self.W_kv_down(x)  # (B, T, rank)

        if kv_cache is not None:
            kv_compressed = torch.cat([kv_cache, kv_compressed], dim=1)

        # Expand back for attention computation
        k = self.W_k_up(kv_compressed).view(B, -1, self.n_heads, self.head_dim).transpose(1, 2)
        v = self.W_v_up(kv_compressed).view(B, -1, self.n_heads, self.head_dim).transpose(1, 2)

        # Standard scaled dot-product attention
        scores = torch.matmul(q, k.transpose(-2, -1)) / (self.head_dim ** 0.5)
        attn = torch.softmax(scores, dim=-1)
        out = torch.matmul(attn, v).transpose(1, 2).reshape(B, T, D)

        # Cache only the compressed representation (rank << d_model)
        return self.W_o(out), kv_compressed
```

Memory savings: Cache stores `(B, T, rank)` instead of `(B, T, n_heads, 2, head_dim)`. With rank=64 vs. 8 heads x 2 x 32 = 512, this is an 8x reduction.

**Example 2: Benchmarking token pruning methods on a video recommendation dataset**

User: "I want to compare H2O and SnapKV for compressing user watch histories in my video recommender. Sequences are 1024 items long."

Approach:
1. Implement both pruning strategies as drop-in cache managers.
2. Test at retention ratios of 128, 256, and 512 tokens out of 1024.
3. Evaluate on GAUC and peak memory.

```python
class H2OPruner:
    """Heavy-Hitter Oracle: retain tokens with highest cumulative attention."""
    def __init__(self, budget: int):
        self.budget = budget

    def prune(self, keys, values, attn_scores):
        # attn_scores: (B, H, T, T) - cumulative attention received
        # Sum attention each token receives across all queries and heads
        token_importance = attn_scores.sum(dim=(1, 2))  # (B, T)
        _, top_indices = token_importance.topk(self.budget, dim=-1)
        top_indices = top_indices.sort(dim=-1).values  # Preserve order

        B = keys.shape[0]
        batch_idx = torch.arange(B).unsqueeze(1).expand_as(top_indices)
        return keys[batch_idx, top_indices], values[batch_idx, top_indices]


class SnapKVPruner:
    """SnapKV: retain tokens with stable high attention across recent queries."""
    def __init__(self, budget: int, window: int = 32):
        self.budget = budget
        self.window = window

    def prune(self, keys, values, attn_scores):
        # Use only recent query window to judge importance
        recent_attn = attn_scores[:, :, -self.window:, :]  # (B, H, W, T)
        # Tokens that are persistently attended (high min across window)
        persistence = recent_attn.min(dim=2).values.sum(dim=1)  # (B, T)
        _, top_indices = persistence.topk(self.budget, dim=-1)
        top_indices = top_indices.sort(dim=-1).values

        B = keys.shape[0]
        batch_idx = torch.arange(B).unsqueeze(1).expand_as(top_indices)
        return keys[batch_idx, top_indices], values[batch_idx, top_indices]
```

Expected results (based on MALLOC findings):
| Method    | Retention | GAUC  | Memory (MB) |
|-----------|-----------|-------|-------------|
| Native    | 1024/1024 | 0.691 | 127.88      |
| H2O       | 256/1024  | 0.674 | 2.00        |
| SnapKV    | 256/1024  | 0.679 | 2.00        |
| H2O       | 512/1024  | 0.685 | 4.00        |
| SnapKV    | 512/1024  | 0.688 | 4.00        |

SnapKV typically retains slightly higher accuracy because its persistence criterion is more stable than cumulative attention.

**Example 3: Adding 2-bit quantization to an existing KV cache**

User: "I can't change my model architecture but need to halve KV-cache memory."

Approach:
1. Apply KIVI-style per-channel asymmetric quantization to cached keys and values.
2. Keep the most recent 128 tokens in full precision (IntactKV insight).

```python
class KIVICache:
    """2-bit KV cache quantization with full-precision recent window."""
    def __init__(self, recent_window: int = 128, bits: int = 2):
        self.recent_window = recent_window
        self.bits = bits
        self.qmax = (1 << bits) - 1

    def quantize(self, tensor):
        # Per-channel min/max for asymmetric quantization
        t_min = tensor.amin(dim=-2, keepdim=True)
        t_max = tensor.amax(dim=-2, keepdim=True)
        scale = (t_max - t_min) / self.qmax
        scale = scale.clamp(min=1e-8)
        zero_point = t_min
        quantized = ((tensor - zero_point) / scale).round().clamp(0, self.qmax).to(torch.uint8)
        return quantized, scale, zero_point

    def dequantize(self, quantized, scale, zero_point):
        return quantized.float() * scale + zero_point

    def compress(self, keys, values):
        T = keys.shape[-2]
        if T <= self.recent_window:
            return keys, values, None

        old_k, recent_k = keys[..., :-self.recent_window, :], keys[..., -self.recent_window:, :]
        old_v, recent_v = values[..., :-self.recent_window, :], values[..., -self.recent_window:, :]

        qk, sk, zk = self.quantize(old_k)
        qv, sv, zv = self.quantize(old_v)

        meta = {"qk": qk, "sk": sk, "zk": zk, "qv": qv, "sv": sv, "zv": zv}
        return recent_k, recent_v, meta

    def reconstruct(self, recent_k, recent_v, meta):
        if meta is None:
            return recent_k, recent_v
        old_k = self.dequantize(meta["qk"], meta["sk"], meta["zk"])
        old_v = self.dequantize(meta["qv"], meta["sv"], meta["zv"])
        full_k = torch.cat([old_k, recent_k], dim=-2)
        full_v = torch.cat([old_v, recent_v], dim=-2)
        return full_k, full_v
```

Memory reduction: 2-bit quantization compresses old tokens by 16x (fp32 to 2-bit), while the recent 128 tokens stay at full precision. For a 1024-length sequence with 128 recent tokens: `(896 / 16 + 128) / 1024 = 18%` of original memory.

## Best Practices

- **Do:** Start with head-level compression (GQA or MLA) as the default choice — MALLOC shows these consistently sit on the Pareto frontier with minimal accuracy loss and significant resource savings.
- **Do:** Measure compression impact on tail users with the longest sequences, not just aggregate metrics. Compression disproportionately affects power users.
- **Do:** Combine methods across granularities when a single method is insufficient — e.g., GQA (head-level) + KIVI (precision-level) stack multiplicatively.
- **Do:** Preserve temporal ordering when pruning tokens. Recommendation sequences have strong recency bias; shuffling pruned tokens destroys positional information.
- **Avoid:** Applying aggressive token pruning (retention < 25%) without validating on long-tail item recall — rare items accessed early in a sequence are the first to be evicted.
- **Avoid:** Using Reformer (LSH-based attention) for recommendation — MALLOC shows it increases MACs by 4x over native while providing no memory savings when caching is used.

## Error Handling

- **Accuracy collapse after compression:** If GAUC drops more than 2% from baseline, reduce the compression ratio incrementally. For token pruning, increase budget by 25%. For quantization, move from 2-bit to 4-bit.
- **OOM during compression benchmarking:** Reduce batch size or profile one method at a time. Use `torch.cuda.empty_cache()` between method evaluations.
- **Quantization numerical instability:** If dequantized values produce NaN in attention softmax, add epsilon to scale factors and clamp attention logits before softmax.
- **Inconsistent results across runs:** Use fixed seeds and report 5-fold cross-validation. Small datasets (Amazon-Electronic) show higher variance — prefer longer-sequence datasets (MicroVideo, KuaiVideo) for reliable comparisons.

## Limitations

- MALLOC's evaluation focuses on HSTU as the backbone. Results may not transfer directly to architectures with fundamentally different attention patterns (e.g., graph-based recommenders or cross-attention fusion models).
- The benchmark uses sequences up to 1024 tokens. For extremely long histories (10K+), the relative rankings of methods may shift as token-pruning methods become more competitive.
- Head-level methods (MQA, GQA, MLA) require architectural changes and retraining. They cannot be applied as post-hoc compression to an already-trained model, unlike token pruning and quantization.
- The paper evaluates on three datasets. Domain-specific recommendation tasks (e.g., news, music, conversational) may exhibit different attention patterns that favor different compression strategies.
- MALLOC does not evaluate hybrid approaches that combine methods across granularities, which could yield better Pareto frontiers.

## Reference

**Paper:** [MALLOC: Benchmarking the Memory-aware Long Sequence Compression for Large Sequential Recommendation](https://arxiv.org/abs/2601.20234v2) — Yu et al., 2026. Look for Table 2 (accuracy comparison across 16 methods), Table 3 (resource consumption profiles), and Figure 4 (Pareto frontier visualization) for the key empirical findings that guide method selection.