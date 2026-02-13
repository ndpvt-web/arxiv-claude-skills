---
name: "nwa-mending-spatial-integrity-torn"
description: "Implement spatially-aware vision token pruning for VLMs using the Nüwa two-stage framework: separation, alignment, and aggregation in the vision encoder plus text-guided pruning in the LLM. Use when asked about 'VLM token pruning', 'vision token reduction', 'spatial-aware pruning', 'visual grounding acceleration', 'efficient VLM inference', or 'preserving spatial layout during token pruning'."
---

# Nüwa: Spatially-Aware Vision Token Pruning for VLMs

This skill enables Claude to implement the Nüwa two-stage token pruning framework for Vision Language Models (VLMs). Nüwa solves a critical problem: existing token pruning methods (FastV, VisionZip, SparseVLM) that use global semantic similarity or attention scores destroy the spatial reference frame encoded in token positions, causing catastrophic degradation on visual grounding tasks (locating objects by coordinates). Nüwa preserves spatial integrity through grid-partitioned aggregation inspired by swarm intelligence, achieving 95% VQA performance retention while improving visual grounding from 7% to 47% of baseline at aggressive pruning rates.

## When to Use

- When implementing vision token pruning for a VLM pipeline and the user needs to preserve spatial localization ability (e.g., bounding box prediction, region-based QA)
- When the user asks to accelerate VLM inference (reduce prefill time, lower TFLOPs) while maintaining both VQA and visual grounding performance
- When building a token reduction module that sits between a vision encoder (e.g., SigLIP, CLIP-ViT) and a language model (e.g., LLaMA, Vicuna)
- When debugging why an existing pruning approach (FastV, FitPrune, VisionZip) causes spatial/grounding tasks to collapse
- When the user needs to implement position embedding strategies for pruned token sequences (RPME vs PERC vs PESP)
- When implementing swarm-intelligence-inspired feature aggregation over grid-structured visual tokens

## Key Technique

**The Problem:** VLMs encode images as grids of visual tokens (e.g., 729 tokens for a 27x27 patch grid). Pruning methods that rank tokens by CLS attention or semantic similarity to text discard tokens uniformly across the spatial layout, destroying the global coordinate system. This is acceptable for VQA ("What color is the car?") but devastating for visual grounding ("Where is the cat?") because grounding requires the model to output spatial coordinates derived from token positions.

**Stage 1 — Separation, Alignment, Aggregation (Vision Encoder):** After the vision encoder, tokens are partitioned into an M x M grid of non-overlapping regions (Separation). Within each region, a composite salience score `S(t_i) = alpha_cls_i * ||k_i||_2` identifies anchor tokens by combining CLS attention weight with key-vector L2 norm (Alignment). Tokens are then classified as Pillar tokens (top quartile by key norm — preserved intact as spatial register anchors) or Collector tokens (which absorb features from spatial neighbors). Aggregation uses weights `W_ij = A_ij * P_ij` combining semantic similarity (ReLU-clamped cosine sim of value vectors) with a spatial proximity gate that zeroes out contributions beyond a distance threshold of 26% of maximum inter-token distance. This produces a reduced set of tokens that retain both informational richness and spatial topology.

**Stage 2 — Text-Guided Pruning (LLM Interior):** After multimodal projection aligns visual tokens into the LLM's embedding space, a mean text query vector `q_bar = (1/K) * sum(q_k)` is computed. Each surviving visual token is scored by relevance `R_i = sim(proj(v_i'), q_bar)`, and only the top-K tokens proceed to deeper LLM layers. This task-conditioned pruning eliminates tokens irrelevant to the specific question while the spatial anchors from Stage 1 ensure the remaining tokens still encode meaningful positions. A Relative Position Mapping Extension (RPME) linearly remaps pruned token positions to span the original coordinate range, preserving relative distances (yielding 5.6-13.4% grounding improvement over alternatives).

## Step-by-Step Workflow

1. **Receive the vision token grid from the encoder.** After the vision encoder (e.g., SigLIP) processes the image, obtain the full token sequence of shape `(N, D)` where N is the number of patch tokens (e.g., 729 for 27x27) and D is the hidden dimension. Also retain each token's 2D grid position `(row, col)`.

2. **Partition into M x M spatial regions (Separation).** Divide the token grid into M x M non-overlapping rectangular regions. Each region contains approximately `N / M^2` tokens. Choose M so that each region has enough tokens for meaningful aggregation (e.g., M=3 for a 27x27 grid yields 9x9 token regions).

3. **Compute composite salience scores (Alignment).** For each token `t_i`, calculate `S(t_i) = alpha_cls_i * ||k_i||_2` where `alpha_cls_i` is the attention weight from the CLS token to `t_i` in the final encoder layer, and `||k_i||_2` is the L2 norm of the token's key vector. This dual signal avoids relying solely on attention (which becomes sparse in deep layers).

4. **Classify tokens as Pillar or Collector.** Rank all tokens by `||k_i||_2`. Tokens in the top 25% (quartile) become Pillar tokens — they are preserved unmodified and act as spatial register anchors. Remaining tokens become Collector tokens, which will aggregate neighbor features.

5. **Aggregate features with spatial proximity gating.** For each Collector token `c_j`, compute aggregation weights from neighboring tokens: `W_ij = A_ij * P_ij` where `A_ij = ReLU(cosine_sim(v_i, v_j))` (semantic similarity of value vectors) and `P_ij = 1 if dist(i,j) < tau else 0` with `tau = 0.26 * max_distance`. Update each Collector's feature as the weighted sum of its neighbors within the same region. Pillar tokens do NOT contribute their features to aggregation and remain intact.

6. **Select spatial anchor tokens per region.** Within each region, select the top-k tokens by salience score S. These become the surviving tokens from Stage 1. The total surviving count is `M^2 * k` plus Pillar tokens that were already preserved.

7. **Project surviving tokens into LLM embedding space.** Pass the reduced token set through the multimodal projector (e.g., MLP or cross-attention bridge) to align visual features with the LLM's text embedding space.

8. **Compute text-guided relevance scores (Stage 2).** After the first few LLM layers process the multimodal sequence, compute the mean text query vector `q_bar` from the text token hidden states. Score each visual token: `R_i = cosine_sim(proj(v_i'), q_bar)`. Retain only the top-K_final tokens by relevance.

9. **Apply Relative Position Mapping Extension (RPME).** For the surviving token set, linearly remap their position IDs so that the minimum maps to position 0 and the maximum maps to the original sequence length. This preserves relative spatial distances and prevents the position embedding from collapsing into a narrow range.

10. **Feed the pruned, spatially-intact token sequence to remaining LLM layers.** The final token set (typically 64-128 tokens from an original 729) retains spatial topology for grounding and task-relevant semantics for VQA, while reducing TFLOPs by ~89% and prefill time by ~62%.

## Concrete Examples

**Example 1: Implementing Stage 1 aggregation in PyTorch**

User: "I have a VLM with a SigLIP encoder outputting 729 visual tokens. I want to reduce them to ~80 tokens while keeping spatial info for grounding tasks. Can you implement the Nüwa Stage 1 pruning?"

Approach:
1. Partition the 27x27 token grid into a 3x3 macro-grid (9 regions of 9x9 = 81 tokens each)
2. Compute salience and classify Pillar vs Collector tokens
3. Aggregate Collector features using proximity-gated semantic similarity
4. Select top-k anchors per region

```python
import torch
import torch.nn.functional as F

def nuwa_stage1(
    tokens: torch.Tensor,       # (B, 729, D) visual token features
    keys: torch.Tensor,          # (B, 729, D_k) key vectors from last encoder layer
    cls_attn: torch.Tensor,      # (B, 729) CLS attention weights
    grid_size: int = 27,         # sqrt(num_tokens)
    M: int = 3,                  # macro-grid partitions per side
    anchors_per_region: int = 8, # tokens to keep per region
    pillar_quantile: float = 0.75,
    tau_ratio: float = 0.26,
):
    B, N, D = tokens.shape
    region_size = grid_size // M  # 9 tokens per side per region

    # Compute 2D positions for each token
    rows = torch.arange(grid_size, device=tokens.device)
    cols = torch.arange(grid_size, device=tokens.device)
    grid_r, grid_c = torch.meshgrid(rows, cols, indexing="ij")
    positions = torch.stack([grid_r.flatten(), grid_c.flatten()], dim=-1).float()  # (N, 2)

    # Compute salience: CLS_attn * ||key||_2
    key_norms = keys.norm(dim=-1)  # (B, 729)
    salience = cls_attn * key_norms  # (B, 729)

    # Classify Pillar tokens (top 25% by key norm)
    pillar_threshold = torch.quantile(key_norms, pillar_quantile, dim=-1, keepdim=True)
    is_pillar = key_norms >= pillar_threshold  # (B, 729)

    # Compute pairwise spatial distances
    dists = torch.cdist(positions.unsqueeze(0), positions.unsqueeze(0)).squeeze(0)  # (N, N)
    tau = tau_ratio * dists.max()
    proximity_mask = (dists < tau).float()  # (N, N)

    # Semantic similarity (value-space, ReLU-clamped)
    token_norm = F.normalize(tokens, dim=-1)
    sim_matrix = torch.bmm(token_norm, token_norm.transpose(1, 2))  # (B, N, N)
    sim_matrix = F.relu(sim_matrix)

    # Aggregation weights: similarity * proximity (Collectors only)
    agg_weights = sim_matrix * proximity_mask.unsqueeze(0)  # (B, N, N)

    # Zero out contributions FROM Pillar tokens (they don't donate features)
    pillar_mask = is_pillar.unsqueeze(1).expand_as(agg_weights)
    agg_weights = agg_weights.masked_fill(pillar_mask, 0)

    # Normalize aggregation weights
    agg_weights = agg_weights / (agg_weights.sum(dim=-1, keepdim=True) + 1e-8)

    # Aggregate: Collector tokens absorb neighbor features
    aggregated = torch.bmm(agg_weights, tokens)  # (B, N, D)
    # Pillar tokens keep original features
    tokens_out = torch.where(is_pillar.unsqueeze(-1), tokens, aggregated)

    # Select top anchors per region by salience
    selected_indices = []
    for r in range(M):
        for c in range(M):
            r_start, r_end = r * region_size, (r + 1) * region_size
            c_start, c_end = c * region_size, (c + 1) * region_size
            region_mask = (
                (grid_r.flatten() >= r_start) & (grid_r.flatten() < r_end) &
                (grid_c.flatten() >= c_start) & (grid_c.flatten() < c_end)
            )
            region_idx = region_mask.nonzero(as_tuple=True)[0]
            region_salience = salience[:, region_idx]  # (B, region_tokens)
            _, topk_local = region_salience.topk(anchors_per_region, dim=-1)
            topk_global = region_idx[topk_local]  # (B, anchors_per_region)
            selected_indices.append(topk_global)

    selected = torch.cat(selected_indices, dim=-1)  # (B, M*M*anchors_per_region)
    pruned_tokens = torch.gather(
        tokens_out, 1, selected.unsqueeze(-1).expand(-1, -1, D)
    )
    pruned_positions = positions[selected[0]]  # (anchors_total, 2)

    return pruned_tokens, pruned_positions, selected
```

**Example 2: Implementing RPME position remapping**

User: "After pruning tokens, my model's visual grounding accuracy dropped. I think position embeddings are broken. How do I fix this?"

Approach:
1. Diagnose: pruned tokens likely have sparse/gapped position IDs
2. Apply RPME to linearly remap positions to span the original range
3. Preserve relative distances between surviving tokens

```python
def apply_rpme(
    original_position_ids: torch.Tensor,  # (B, K_surviving) — original grid positions
    original_max_pos: int = 728,          # max position in unpruned sequence
):
    """Relative Position Mapping Extension: remap sparse position IDs
    to span [0, original_max_pos] while preserving relative distances."""
    pos_min = original_position_ids.min(dim=-1, keepdim=True).values.float()
    pos_max = original_position_ids.max(dim=-1, keepdim=True).values.float()
    pos_float = original_position_ids.float()

    # Linear remap: new_pos = (pos - min) / (max - min) * original_max_pos
    remapped = (pos_float - pos_min) / (pos_max - pos_min + 1e-8) * original_max_pos
    return remapped.long()
```

**Example 3: Adding text-guided Stage 2 pruning to an existing VLM**

User: "I already have a LLaVA-style model. How do I add Nüwa's Stage 2 text-guided pruning inside the LLM layers?"

Approach:
1. Identify the LLM layer after multimodal alignment (typically layer 1-3)
2. Extract text token hidden states and compute mean query vector
3. Score visual tokens by relevance and keep top-K

```python
def nuwa_stage2(
    hidden_states: torch.Tensor,   # (B, seq_len, D) — full sequence after layer L
    visual_token_mask: torch.Tensor,  # (B, seq_len) bool — which positions are visual
    text_token_mask: torch.Tensor,    # (B, seq_len) bool — which positions are text
    K_final: int = 64,
):
    """Text-guided pruning: retain top-K visual tokens most relevant to the query."""
    # Mean text query vector
    text_hidden = hidden_states * text_token_mask.unsqueeze(-1).float()
    q_bar = text_hidden.sum(dim=1) / text_token_mask.sum(dim=1, keepdim=True).float()  # (B, D)

    # Visual token features
    visual_indices = visual_token_mask.nonzero(as_tuple=False)  # (num_visual, 2)
    visual_hidden = hidden_states[visual_token_mask]  # (total_visual, D)

    # Relevance scores per batch
    B = hidden_states.shape[0]
    kept_indices_list = []
    for b in range(B):
        b_mask = visual_token_mask[b]
        b_visual = hidden_states[b, b_mask]  # (n_vis, D)
        relevance = F.cosine_similarity(b_visual, q_bar[b].unsqueeze(0), dim=-1)
        _, topk = relevance.topk(min(K_final, b_visual.shape[0]))
        vis_positions = b_mask.nonzero(as_tuple=True)[0]
        kept_indices_list.append(vis_positions[topk])

    return kept_indices_list  # List of (K_final,) index tensors per batch
```

## Best Practices

- **Do:** Partition the spatial grid into regions before any pruning. Global ranking destroys spatial coverage — you need at least some surviving tokens in every image quadrant for grounding to work.
- **Do:** Preserve Pillar tokens (high key-norm) intact without aggregation. These act as register tokens that stabilize feature distributions across the sequence.
- **Do:** Use RPME for position remapping after pruning. Naive approaches (keeping original sparse IDs or compressing to a contiguous range) lose either global scale or relative distance information.
- **Do:** Place Stage 2 pruning after at least 1-2 LLM layers so text-visual cross-attention has occurred and relevance scores are meaningful.
- **Avoid:** Using attention scores alone for salience. CLS attention becomes sparse and unreliable in deep encoder layers — always multiply by key norm.
- **Avoid:** Setting the proximity threshold tau too low. The 26% of max distance value was empirically validated; going much lower fragments the aggregation neighborhoods and starves Collector tokens of context.

## Error Handling

- **Salience scores are all near-zero:** This happens when CLS attention is nearly uniform (common in early fine-tuning). Fall back to key-norm-only ranking: `S(t_i) = ||k_i||_2`.
- **Region has fewer tokens than `anchors_per_region`:** Clamp the selection to `min(region_token_count, anchors_per_region)` and redistribute the deficit to neighboring regions.
- **Text-guided pruning removes all tokens from a spatial region:** Add a constraint that at least 1 token per Stage-1 region survives Stage 2. This prevents spatial blind spots in grounding outputs.
- **Position embedding lookup out of range after RPME:** Ensure the remapped IDs are clamped to `[0, max_position_embedding - 1]` and that the position embedding table covers the original sequence length.
- **OOM on pairwise distance/similarity matrices:** For large token counts (>1024), compute aggregation within each region independently rather than over the full NxN matrix.

## Limitations

- **VQA-only pipelines see minimal benefit.** If your application never does visual grounding, bounding box prediction, or spatial reasoning, simpler pruning methods (FastV, FitPrune) are sufficient and easier to implement.
- **Fixed grid partitioning assumes uniform patch layouts.** For models with dynamic resolution or non-square image inputs, the M x M grid needs adaptation (e.g., aspect-ratio-aware partitioning).
- **Stage 1 adds a small computational overhead** (~17.56 MFLOPs for computing salience and aggregation weights), which is negligible for large models but noticeable if you're pruning an already-tiny encoder.
- **Hyperparameters (M, tau, quartile threshold) were tuned on LLaVA-1.5/InternVL.** They may need re-tuning for architecturally different VLMs (e.g., Flamingo-style cross-attention models where visual tokens interact differently).
- **Not applicable to text-only token pruning** or modalities without 2D spatial structure.

## Reference

- **Paper:** [Nüwa: Mending the Spatial Integrity Torn by VLM Token Pruning](https://arxiv.org/abs/2602.02951v1) — Huang et al., 2026. Focus on Section 3 (method) for the separation/alignment/aggregation formulas, Section 4.3 for the RPME ablation, and Table 2 for visual grounding benchmarks showing the 7% to 47% improvement.
- **Code:** https://github.com/Man-PaperRejected/Nuwa