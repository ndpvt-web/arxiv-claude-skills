---
name: "unleashing-potential-sparse-attention"
description: "Implement SparseCTR's three-branch sparse attention for efficient CTR prediction on long user behavior sequences. Use when: 'build a CTR model for long behavior sequences', 'implement sparse attention for recommendation', 'scale user behavior modeling efficiently', 'add temporal encoding to sequential recommendation', 'chunk user histories for parallel attention', 'implement personalized behavior segmentation for ads ranking'."
---

# SparseCTR: Sparse Attention for Long-Term User Behavior CTR Prediction

This skill enables Claude to implement SparseCTR, a sparse self-attention architecture that efficiently models long-term user behavior sequences (1000+ items) for click-through rate prediction. The core technique replaces full O(n^2) self-attention with three specialized sparse branches — global, transition, and local — over personalized time-based chunks, achieving linear-scale attention while capturing multi-granularity user interests. It includes a composite relative temporal encoding that captures both sequential recency and periodic patterns (time-of-day, weekday/weekend) through learnable head-specific bias coefficients.

## When to Use

- When building a CTR prediction model that must process user behavior sequences longer than 256 items efficiently
- When the user wants to add sparse attention to a recommendation or ads ranking system without sacrificing accuracy
- When implementing a user behavior encoder that must capture both long-term stable interests and short-term intent shifts
- When adding time-aware positional encoding to a sequential recommendation model that must understand recency, daily cycles, and weekend effects
- When scaling a Transformer-based recommender and observing quadratic attention costs becoming a bottleneck
- When the user needs a production-ready architecture that demonstrates scaling law behavior (consistent AUC gains across 3 orders of magnitude in FLOPs)

## Key Technique

**Personalized Time-Based Chunking.** Unlike fixed-window chunking, SparseCTR segments each user's behavior sequence by finding the largest time gaps between consecutive actions. Given a desired number of chunks P, the algorithm selects the top-(P-1) largest time intervals as cut points. This produces variable-length chunks where behaviors within a chunk are temporally cohesive (e.g., a single browsing session) while chunk boundaries correspond to natural interest shifts. Padding tokens naturally form their own chunk. All users get the same number of chunks regardless of sequence length, enabling batched parallel processing.

**Three-Branch Sparse Attention.** Each SparseBlock computes three attention patterns in parallel: (1) **Global attention** aggregates keys/values within each chunk into a single summary vector per chunk, then each query attends to these P summary vectors — capturing stable long-term interests at O(n*P) cost. (2) **Transition attention** selects the last m behaviors from each chunk as "tail tokens" representing interest transitions, then each query attends to these m*P tokens — capturing how user preferences shift between sessions. (3) **Local attention** uses a sliding window of width w so each query attends only to its w nearest predecessors plus a compressed user-profile token — capturing immediate short-term intent. The three outputs are fused via a learned gating mechanism: `[alpha1, alpha2, alpha3] = softmax(concat(o_global, o_transition, o_local) @ W_gate)` producing a weighted combination.

**Composite Relative Temporal Encoding.** Instead of standard positional encoding, SparseCTR injects three bias terms directly into attention logits, each with a learnable per-head slope coefficient: (1) **Relative time bias**: `-floor(log2(delta_t)) * s1_h` — log-bucketed recency decay. (2) **Relative hour bias**: `-sin(pi * hour_diff / 24) * s2_h` — cyclic daily pattern (items viewed at similar times of day get higher affinity). (3) **Relative weekend bias**: `{0 if same type, -1 if different} * s3_h` — weekday/weekend alignment. Slopes are initialized as a geometric sequence `(2^(-8/H))^(h-1)` across H heads, giving each head a different temporal sensitivity.

## Step-by-Step Implementation Workflow

1. **Prepare behavior sequences.** For each user, collect their chronological item interaction history as a list of `(item_id, item_features, timestamp)` tuples. Pad or truncate to a fixed max length L (e.g., 1024). Record the timestamp for each position.

2. **Implement TimeChunking.** For each user's sequence, compute time deltas between consecutive items: `delta[i] = timestamp[i] - timestamp[i-1]`. Find the top-(P-1) largest deltas (P = number of chunks, typically 16-32). Create segment IDs via cumulative sum of one-hot markers at cut positions. Extract chunk summary tokens (mean-pool within each chunk) and tail tokens (last m items per chunk, m typically 1-3).

3. **Build the RelTemporal encoder.** Create three bias computation functions that take query and key timestamps and return bias matrices. Initialize per-head slope parameters `s1, s2, s3` as `nn.Parameter` tensors of shape `(num_heads,)` with geometric initialization `(2^(-8/H))^(h-1)`. The biases are added directly to attention logits before softmax.

4. **Implement the three attention branches.** Each branch is a standard multi-head attention but with different key/value sets: Global uses chunk summaries (shape `[B, P, d]`), Transition uses tail tokens (shape `[B, m*P, d]`), Local uses a sliding window mask (shape `[B, L, w+1, d]` including the user-profile token). All three receive the same query tensor and add the RelTemporal biases to their respective attention score matrices.

5. **Fuse branch outputs with gating.** Concatenate the three branch outputs along the feature dimension to get shape `[B, L, 3*d]`. Project through `W_gate` of shape `[3*d, 3]`, apply softmax to get per-position weights `[B, L, 3]`, then compute weighted sum of the three outputs.

6. **Stack SparseBlocks into the encoder.** Each SparseBlock consists of: RMSNorm -> Three-branch EvoAttention -> residual add -> RMSNorm -> SwiGLU FFN (hidden dim = 3*d) -> residual add. Stack 2 blocks (sufficient for most settings; the paper uses 2 layers with d=32 and 8 heads).

7. **Build the prediction head.** Extract the output embeddings at target item positions. Pass through a prediction MLP (e.g., `[d, 1]` with ReLU) to produce logits. Train with binary cross-entropy loss over all candidate items.

8. **Configure hyperparameters.** Start with: embedding dim d=32, num_heads=8, num_layers=2, num_chunks P=16, tail_tokens m=2, local_window w=64, max_seq_len L=1024. These are the paper's validated defaults for industrial-scale datasets.

9. **Validate with scaling tests.** Confirm that AUC improves monotonically as you scale sequence length (128 -> 256 -> 512 -> 1024) and model dimension. The relationship should follow a power law: `AUC(X) = E - A / X^alpha`.

## Concrete Examples

**Example 1: Implementing TimeChunking in PyTorch**

User: "I need to segment user behavior sequences into variable-length chunks based on time gaps for parallel attention processing."

Approach:
1. Compute pairwise time deltas between consecutive items
2. Find top-K largest gaps as segment boundaries
3. Assign segment IDs and extract chunk representations

Output:
```python
import torch
import torch.nn as nn

class TimeChunking(nn.Module):
    def __init__(self, num_chunks: int, tail_tokens_per_chunk: int = 2):
        super().__init__()
        self.num_chunks = num_chunks
        self.m = tail_tokens_per_chunk

    def forward(self, timestamps: torch.Tensor, values: torch.Tensor, mask: torch.Tensor):
        """
        Args:
            timestamps: [B, L] unix timestamps per behavior
            values: [B, L, d] behavior embeddings
            mask: [B, L] boolean, True for valid (non-padding) positions
        Returns:
            chunk_summaries: [B, P, d] mean-pooled per-chunk representations
            tail_tokens: [B, P*m, d] last m tokens from each chunk
            segment_ids: [B, L] integer chunk assignment per position
        """
        B, L, d = values.shape
        P = self.num_chunks

        # Compute time deltas between consecutive behaviors
        deltas = timestamps[:, 1:] - timestamps[:, :-1]  # [B, L-1]
        deltas = deltas * mask[:, 1:]  # zero out padding deltas

        # Find top-(P-1) largest time gaps as cut points
        _, cut_indices = deltas.topk(P - 1, dim=-1)  # [B, P-1]
        cut_indices = cut_indices + 1  # shift to mark start of new chunk

        # Build segment IDs via scatter + cumsum
        cuts = torch.zeros(B, L, device=values.device)
        cuts.scatter_(1, cut_indices, 1.0)
        segment_ids = cuts.cumsum(dim=1).long()  # [B, L]

        # Mean-pool within each chunk for global attention keys
        chunk_summaries = torch.zeros(B, P, d, device=values.device)
        chunk_counts = torch.zeros(B, P, 1, device=values.device)
        seg_expanded = segment_ids.unsqueeze(-1).expand_as(values)
        chunk_summaries.scatter_add_(1, seg_expanded, values * mask.unsqueeze(-1))
        chunk_counts.scatter_add_(1, segment_ids.unsqueeze(-1),
                                   mask.unsqueeze(-1).float())
        chunk_summaries = chunk_summaries / chunk_counts.clamp(min=1)

        # Extract tail tokens (last m valid tokens per chunk)
        # Implementation: for each chunk, find the last m positions
        tail_tokens = self._extract_tails(values, segment_ids, mask, P)
        return chunk_summaries, tail_tokens, segment_ids

    def _extract_tails(self, values, segment_ids, mask, P):
        B, L, d = values.shape
        tails = []
        for p in range(P):
            chunk_mask = (segment_ids == p) & mask  # [B, L]
            # Reverse cumsum to find last m positions
            rev_cumsum = chunk_mask.flip(1).cumsum(1).flip(1)
            tail_mask = (rev_cumsum <= self.m) & chunk_mask
            # Gather with masking (simplified: take weighted sum)
            tail_vals = values * tail_mask.unsqueeze(-1)
            # Pack into fixed m slots per chunk
            tails.append(tail_vals.sum(dim=1, keepdim=True) / tail_mask.sum(1, keepdim=True).clamp(min=1).unsqueeze(-1))
        return torch.cat(tails, dim=1)  # [B, P, d]
```

**Example 2: Implementing Composite Relative Temporal Encoding**

User: "I want time-aware attention biases that capture recency, daily patterns, and weekend effects."

Approach:
1. Compute pairwise time differences between all query-key pairs
2. Apply log-bucketed recency, sinusoidal hour-of-day, and weekend match biases
3. Scale each by learnable per-head coefficients

Output:
```python
import torch
import torch.nn as nn
import math

class RelTemporalBias(nn.Module):
    def __init__(self, num_heads: int):
        super().__init__()
        H = num_heads
        # Initialize slopes as geometric sequence
        base = 2 ** (-8.0 / H)
        init_slopes = torch.tensor([base ** (h) for h in range(H)])

        self.s_time = nn.Parameter(init_slopes.clone())    # recency slopes
        self.s_hour = nn.Parameter(init_slopes.clone())    # daily cycle slopes
        self.s_weekend = nn.Parameter(init_slopes.clone()) # weekend slopes

    def forward(self, q_times: torch.Tensor, k_times: torch.Tensor):
        """
        Args:
            q_times: [B, Lq] timestamps for queries
            k_times: [B, Lk] timestamps for keys
        Returns:
            bias: [B, H, Lq, Lk] additive attention bias
        """
        # Pairwise time differences in seconds
        dt = (q_times.unsqueeze(-1) - k_times.unsqueeze(-2)).float()  # [B, Lq, Lk]

        # 1. Relative time bias: log-bucketed recency
        dt_abs = dt.abs().clamp(min=1)
        time_bias = -torch.floor(torch.log2(dt_abs))  # [B, Lq, Lk]
        time_bias = time_bias.unsqueeze(1) * self.s_time[None, :, None, None]

        # 2. Relative hour bias: cyclic daily distance
        dt_hours = (dt.abs() / 3600.0) % 24  # hours difference mod 24
        hour_bias = -torch.sin(math.pi * dt_hours / 24.0)  # [B, Lq, Lk]
        hour_bias = hour_bias.unsqueeze(1) * self.s_hour[None, :, None, None]

        # 3. Relative weekend bias: same/different day type
        q_dow = ((q_times / 86400).long() % 7)  # day of week
        k_dow = ((k_times / 86400).long() % 7)
        q_weekend = (q_dow >= 5).float()   # [B, Lq]
        k_weekend = (k_dow >= 5).float()   # [B, Lk]
        weekend_match = (q_weekend.unsqueeze(-1) - k_weekend.unsqueeze(-2)).abs()
        weekend_bias = -weekend_match  # 0 if same, -1 if different
        weekend_bias = weekend_bias.unsqueeze(1) * self.s_weekend[None, :, None, None]

        return time_bias + hour_bias + weekend_bias  # [B, H, Lq, Lk]
```

**Example 3: Full SparseBlock with Three-Branch Attention and Gated Fusion**

User: "Build the core SparseCTR attention block that combines global, transition, and local attention with gating."

Approach:
1. Implement three parallel attention computations over different key/value sets
2. Add temporal biases to each branch's attention logits
3. Fuse outputs with learned gating weights

Output:
```python
class SparseBlock(nn.Module):
    def __init__(self, d_model: int = 32, num_heads: int = 8,
                 local_window: int = 64, ffn_mult: int = 3):
        super().__init__()
        self.d_model = d_model
        self.num_heads = num_heads
        self.head_dim = d_model // num_heads
        self.w = local_window

        # Shared Q projection, separate K/V for each branch
        self.W_q = nn.Linear(d_model, d_model)
        self.W_k = nn.Linear(d_model, d_model)
        self.W_v = nn.Linear(d_model, d_model)
        self.W_o = nn.Linear(d_model, d_model)

        # Gating: maps concatenated 3-branch output to 3 weights
        self.W_gate = nn.Linear(3 * d_model, 3)

        # Temporal bias
        self.temporal = RelTemporalBias(num_heads)

        # RMSNorm + SwiGLU FFN
        self.norm1 = nn.RMSNorm(d_model)
        self.norm2 = nn.RMSNorm(d_model)
        self.ffn_gate = nn.Linear(d_model, ffn_mult * d_model)
        self.ffn_up = nn.Linear(d_model, ffn_mult * d_model)
        self.ffn_down = nn.Linear(ffn_mult * d_model, d_model)

    def _attention(self, Q, K, V, bias=None, mask=None):
        scale = self.head_dim ** -0.5
        scores = torch.matmul(Q, K.transpose(-2, -1)) * scale
        if bias is not None:
            scores = scores + bias
        if mask is not None:
            scores = scores.masked_fill(~mask, float('-inf'))
        return torch.matmul(torch.softmax(scores, dim=-1), V)

    def forward(self, x, chunk_summaries, tail_tokens,
                q_times, k_times_global, k_times_transition, mask):
        B, L, d = x.shape
        H, hd = self.num_heads, self.head_dim
        residual = x
        x = self.norm1(x)

        Q = self.W_q(x).view(B, L, H, hd).transpose(1, 2)
        K = self.W_k(x).view(B, L, H, hd).transpose(1, 2)
        V = self.W_v(x).view(B, L, H, hd).transpose(1, 2)

        # Branch 1: Global — attend to chunk summaries
        K_g = self.W_k(chunk_summaries).view(B, -1, H, hd).transpose(1, 2)
        V_g = self.W_v(chunk_summaries).view(B, -1, H, hd).transpose(1, 2)
        bias_g = self.temporal(q_times, k_times_global)
        o_global = self._attention(Q, K_g, V_g, bias=bias_g)

        # Branch 2: Transition — attend to tail tokens
        K_t = self.W_k(tail_tokens).view(B, -1, H, hd).transpose(1, 2)
        V_t = self.W_v(tail_tokens).view(B, -1, H, hd).transpose(1, 2)
        bias_t = self.temporal(q_times, k_times_transition)
        o_trans = self._attention(Q, K_t, V_t, bias=bias_t)

        # Branch 3: Local — sliding window of width w
        # Use causal local mask for efficiency
        o_local = self._local_attention(Q, K, V, q_times)

        # Gated fusion
        o_g = o_global.transpose(1, 2).reshape(B, L, d)
        o_t = o_trans.transpose(1, 2).reshape(B, L, d)
        o_l = o_local.transpose(1, 2).reshape(B, L, d)
        gate_input = torch.cat([o_g, o_t, o_l], dim=-1)
        gates = torch.softmax(self.W_gate(gate_input), dim=-1)  # [B, L, 3]
        fused = (gates[..., 0:1] * o_g +
                 gates[..., 1:2] * o_t +
                 gates[..., 2:3] * o_l)
        fused = self.W_o(fused) + residual

        # SwiGLU FFN
        residual = fused
        fused = self.norm2(fused)
        fused = self.ffn_down(
            nn.functional.silu(self.ffn_gate(fused)) * self.ffn_up(fused)
        ) + residual
        return fused

    def _local_attention(self, Q, K, V, q_times):
        """Sliding window attention with window size w."""
        B, H, L, hd = Q.shape
        # Create causal local attention mask
        positions = torch.arange(L, device=Q.device)
        dist = positions.unsqueeze(0) - positions.unsqueeze(1)  # [L, L]
        local_mask = (dist >= 0) & (dist < self.w)  # [L, L]
        local_mask = local_mask.unsqueeze(0).unsqueeze(0)  # [1, 1, L, L]

        bias_l = self.temporal(q_times, q_times)
        return self._attention(Q, K, V, bias=bias_l, mask=local_mask)
```

## Best Practices

- **Do:** Initialize temporal slope parameters with the geometric sequence `(2^(-8/H))^(h-1)`. This gives each attention head a distinct temporal scale — some heads focus on recent behaviors, others span the full history. The paper's ablation shows this initialization matters significantly.
- **Do:** Use log-bucketed time differences (`floor(log2(dt))`) rather than raw time deltas. Raw timestamps span orders of magnitude and destabilize attention; log-bucketing compresses the range while preserving relative ordering.
- **Do:** Set the number of chunks P proportional to expected distinct browsing sessions (16-32 for 1024-length sequences). Too few chunks collapse distinct interests; too many create single-item chunks that lose the summarization benefit.
- **Do:** Include the user-profile token (compressed user features) as an extra key in local attention. This grounds short-term attention in the user's static attributes.
- **Avoid:** Using fixed-width chunking instead of time-gap-based chunking. Fixed chunks split browsing sessions arbitrarily and lose the "high cohesion within chunks" property that makes global attention meaningful.
- **Avoid:** Dropping any of the three temporal bias components without ablation. The relative time bias contributes most (-0.068 AUC when removed), but hour and weekend biases provide meaningful gains on datasets with daily/weekly patterns.

## Error Handling

- **Empty chunks:** When a user has fewer interactions than the number of chunks, some chunks will be empty. Zero-pad chunk summaries and mask them in global attention to prevent NaN from division by zero in mean-pooling.
- **Identical timestamps:** If consecutive behaviors share the same timestamp (batch actions), `log2(0)` is undefined. Clamp `delta_t` to a minimum of 1 second before log-bucketing.
- **Short sequences:** For sequences shorter than the local window w, local attention degenerates to full attention within the sequence — this is correct behavior and needs no special handling.
- **Numerical stability in gating:** The softmax over gate logits can saturate if one branch dominates. Monitor gate weight distributions during training; if one branch consistently gets >0.9 weight, reduce its learning rate or add entropy regularization.
- **Variable sequence lengths in batching:** Use attention masks consistently across all three branches. Global and transition attention must mask out chunks derived from padding regions.

## Limitations

- **Cold-start users:** SparseCTR requires meaningful behavior history (50+ items) for the chunking and multi-branch attention to provide value over simpler models. For new users with <10 interactions, a simple DNN over available features is likely sufficient.
- **Non-temporal domains:** The composite temporal encoding assumes timestamped sequential behaviors. For domains without meaningful temporal signals (e.g., static attribute prediction), the temporal bias components add parameters without benefit — use standard positional encoding instead.
- **Chunk count sensitivity:** The number of chunks P is a fixed hyperparameter that implicitly assumes a typical number of interest segments per user. Datasets where users have highly variable interest diversity may need per-cohort tuning of P.
- **Memory for tail token extraction:** The current tail-token extraction requires iterating over chunks, which can be slow for very large P. For P > 64, consider approximate methods or pre-computing tail indices offline.
- **Training paradigm:** The paper trains for a single epoch on large-scale industrial data. On smaller academic datasets, multi-epoch training with learning rate scheduling may be necessary, and the scaling law behavior may not manifest until sequence lengths exceed 256.

## Reference

**Paper:** "Unleashing the Potential of Sparse Attention on Long-term Behaviors for CTR Prediction" (WWW '26)
[arXiv:2601.17836](https://arxiv.org/abs/2601.17836v1) — Focus on Section 3 (Method) for the chunking algorithm, three-branch attention formulas, and temporal encoding equations; Section 4.4 for ablation results showing component contributions.
**Code:** [github.com/laiweijiang/SparseCTR](https://github.com/laiweijiang/SparseCTR) — Core implementation in `handle_layer/handle_lib/handle_rec_unit.py`.