---
name: "topt-task-oriented-prompt-tuning"
description: "Apply Task-Oriented Prompt Tuning (ToPT) to build urban region representation learning pipelines that combine spatial-aware embeddings with LLM-driven task-specific prompting. Use when: 'build a crime prediction model from urban data', 'create region embeddings from POI and mobility data', 'design a task-aware urban computing pipeline', 'align spatial embeddings with downstream objectives', 'fuse multi-modal urban data with spatial priors', 'prompt-tune region representations for prediction tasks'."
---

# ToPT: Task-Oriented Prompt Tuning for Urban Region Representation Learning

This skill enables Claude to design and implement two-stage urban computing pipelines based on the ToPT framework. The core idea: instead of learning generic region embeddings and hoping they transfer to downstream tasks, ToPT injects spatial structure into multi-source data fusion (Stage 1) and then aligns those embeddings with task-specific semantics extracted from a frozen multimodal LLM via cross-attention (Stage 2). This produces region representations that are both spatially coherent and tuned to the prediction objective — achieving up to 64% improvement over task-agnostic baselines on urban prediction benchmarks.

## When to Use

- When the user needs to predict urban phenomena (crime rates, check-ins, service demand, land use) from heterogeneous region data
- When building a pipeline that fuses POI distributions, mobility flows, satellite imagery, and street-view photos into region embeddings
- When the user wants spatial-aware graph neural network architectures that respect geographic distance and centrality
- When adapting pre-trained region embeddings to a specific downstream task without full retraining
- When the user asks how to use LLM-generated semantic vectors as soft prompts for non-NLP prediction tasks
- When designing multi-view urban data fusion with explicit spatial bias injection

## Key Technique

**Stage 1 — Spatial-Aware Region Embedding Learning (SREL):** Multiple urban data views (POI categories, land-use labels, human mobility matrices) are each encoded into view-specific embeddings, then fused via a Graphormer-based module. The key innovation is injecting spatial priors as learnable attention biases: a distance-based adjacency matrix A defines which regions interact, and region centrality scores (c_i = sum of A_ij, normalized) are added as positional features. The self-attention computes `alpha_ij = softmax((Q_i * K_j^T)/sqrt(d) + B_ij)` where B_ij is derived from A, forcing the model to weight nearby and well-connected regions more strongly. This produces spatially coherent embeddings E in R^(N x d).

**Stage 2 — Task-Oriented Prompting (Prompt4RE):** A frozen multimodal LLM (e.g., a vision-language model) receives task-specific templates combined with satellite imagery, street-view photos, and geo-text for each region. The LLM's last-layer hidden states become prompt vectors P. Multi-head cross-attention then aligns E (as queries) with P (as keys/values): `att = softmax(E W_Q (P W_K)^T / sqrt(d)) P W_V`. After residual connections and layer normalization, the output is projected into soft prompts S, concatenated with E to form `F = [E || S]` in R^(N x 2d). A task-specific prediction head (feedforward network) maps F to the target variable. Only the cross-attention and prediction head are trained; the LLM stays frozen.

**Why this matters:** The spatial bias injection prevents the model from treating distant regions as equivalent (a common failure mode), while the cross-attention alignment lets the same base embeddings adapt to completely different tasks (crime vs. check-ins vs. land use) by changing only the prompt template and lightweight alignment layers.

## Step-by-Step Workflow

1. **Partition the study area into regions.** Define a grid or use census tracts/TAZs. Each region i gets a unique ID and centroid coordinates. Compute a distance-based adjacency matrix A where A_ij = 1 if distance(i,j) < threshold, else 0.

2. **Extract multi-view features per region.** For each region, collect: (a) POI category distribution vector X_p (e.g., 14-category histogram from OpenStreetMap), (b) land-use category vector X_l, (c) mobility flow vector X_m (incoming/outgoing trip counts from taxi or transit data). Normalize each view independently.

3. **Encode each view into embeddings.** Apply a view-specific encoder (linear projection or small MLP) to produce Z_p, Z_l, Z_m each in R^(N x d). Concatenate or sum them into a fused matrix H_tilde.

4. **Inject spatial priors.** Compute region centrality c_i = sum_j(A_ij), max-normalize, project via a learned linear layer, and add to H_tilde to get H'. Construct spatial bias matrix B from A (e.g., B_ij = learned_embedding[A_ij]) for use in attention.

5. **Apply Graphormer-based fusion.** Run L layers (2-5 depending on task complexity) of multi-head self-attention over H' with the additive spatial bias B in the attention logits. This yields the spatial-aware embeddings E in R^(N x d). Train this stage end-to-end with the downstream loss (MSE for regression tasks) for ~4000 epochs.

6. **Construct task-specific prompt templates.** Write natural-language templates that describe the prediction objective and regional context. Example for crime prediction: "Given the satellite image of this urban region at coordinates ({lat}, {lon}) with nearby POIs: {poi_list}, assess the factors that may influence local crime rates." Pair with satellite/street-view imagery per region.

7. **Generate prompt vectors from a frozen MLLM.** Feed each region's template + images into a frozen multimodal LLM (e.g., LLaVA, InternVL, or GPT-4V via API). Extract last-layer hidden states as prompt vectors P in R^(N x d_llm). Cache these — they don't change during training.

8. **Align embeddings with prompts via cross-attention.** Implement multi-head cross-attention with E as queries and P as keys/values. Apply residual connection with a learned projection W_res, then layer normalization. Project the output to soft prompts S in R^(N x d). Concatenate: F = [E || S].

9. **Train the task-specific prediction head.** Attach a 2-3 layer feedforward network to F. Train only the cross-attention parameters and the prediction head for ~3000 epochs with MSE loss, learning rate 1e-4 to 3e-4. The MLLM and SREL encoder stay frozen.

10. **Evaluate and iterate.** Report MAE, RMSE, and R-squared. Run ablations: remove spatial bias (B_ij = 0), remove prompt alignment (S = 0), or swap prompt templates to confirm task-specificity. Use 5 random seeds and report mean/std.

## Concrete Examples

**Example 1: Crime prediction pipeline for Chicago**

User: "I have Chicago POI data, taxi trip records, and satellite imagery for 180 community areas. Build a crime rate prediction model."

Approach:
1. Load POI data; compute 14-category distribution per region (food, retail, education, etc.)
2. Load taxi OD matrix; compute inflow/outflow vectors per region
3. Build adjacency matrix: A_ij = 1 if centroids within 3km
4. Encode POI and mobility views with 2-layer MLPs into 144-dim embeddings each
5. Fuse with spatial-biased Graphormer (3 layers, 4 heads, lambda=0.2)
6. Generate prompt vectors: feed satellite images + template "Analyze urban safety factors for region at ({lat},{lon}) with POIs: {top_5_poi_categories}" into frozen LLaVA-7B
7. Cross-attention alignment (4 heads), concatenate to get 288-dim region features
8. Train 2-layer FNN (288 -> 128 -> 1) with MSE loss

Output:
```
Model: ToPT-Crime
Regions: 180 | Embedding dim: 144 | Final dim: 288
Stage 1 (SREL): 4000 epochs, lr=0.0005, 3 Graphormer layers
Stage 2 (Prompt4RE): 3000 epochs, lr=0.0002
Results (5-run avg):
  MAE:  49.3 (+/- 1.2)
  RMSE: 62.7 (+/- 1.8)
  R2:   0.874 (+/- 0.008)
```

**Example 2: Adapting existing region embeddings to a new task**

User: "I already have pre-trained region embeddings from a mobility model. How do I adapt them for check-in prediction without retraining from scratch?"

Approach:
1. Freeze the existing embeddings E (N x d)
2. Collect street-view images and POI text for each region
3. Write a check-in-specific template: "Given the street scene and local businesses ({poi_list}) at ({lat},{lon}), what factors drive visitor check-in frequency in this area?"
4. Run through frozen MLLM to get prompt vectors P
5. Train only the cross-attention alignment layer + prediction head
6. Compare performance with and without the prompt alignment

Output:
```python
# Minimal Prompt4RE adaptation layer
class Prompt4RE(nn.Module):
    def __init__(self, embed_dim=144, prompt_dim=4096, n_heads=4):
        super().__init__()
        self.cross_attn = nn.MultiheadAttention(
            embed_dim, n_heads, kdim=prompt_dim, vdim=prompt_dim, batch_first=True
        )
        self.W_res = nn.Linear(embed_dim, embed_dim)
        self.norm = nn.LayerNorm(embed_dim)
        self.proj = nn.Linear(embed_dim, embed_dim)

    def forward(self, E, P):
        # E: (N, d), P: (N, d_llm) -> unsqueeze for MHA
        attn_out, _ = self.cross_attn(
            E.unsqueeze(0), P.unsqueeze(0), P.unsqueeze(0)
        )
        aligned = self.norm(attn_out.squeeze(0) + self.W_res(E))
        S = self.proj(aligned)
        return torch.cat([E, S], dim=-1)  # (N, 2d)
```

**Example 3: Designing spatial bias for a custom city grid**

User: "I'm working with a 20x20 grid over downtown Seattle. How do I implement the spatial attention bias?"

Approach:
1. Compute pairwise distances between grid cell centroids
2. Threshold to get adjacency matrix A (e.g., adjacent = sharing an edge or corner)
3. Compute centrality and spatial bias embeddings
4. Inject into Graphormer attention

Output:
```python
import numpy as np
import torch
import torch.nn as nn

def build_spatial_components(grid_size=20, threshold_km=1.5):
    N = grid_size * grid_size
    coords = np.array([(i, j) for i in range(grid_size) for j in range(grid_size)])

    # Distance-based adjacency (using grid units; scale to km as needed)
    dist = np.linalg.norm(coords[:, None] - coords[None, :], axis=-1)
    A = (dist <= np.sqrt(2)).astype(float)  # 8-connected neighbors
    np.fill_diagonal(A, 0)

    # Region centrality: normalized degree
    centrality = A.sum(axis=1)
    centrality = centrality / centrality.max()

    # Spatial bias: learnable embedding indexed by adjacency
    # A_ij in {0, 1} -> 2 possible values
    spatial_bias_embed = nn.Embedding(2, 1)  # learned scalar per relationship

    A_tensor = torch.tensor(A, dtype=torch.long)
    B = spatial_bias_embed(A_tensor).squeeze(-1)  # (N, N)

    return torch.tensor(centrality, dtype=torch.float32), B, A_tensor

# Usage in attention:
# attn_logits = (Q @ K.T) / sqrt(d) + lambda * B
```

## Best Practices

- **Do:** Cache MLLM prompt vectors after generation. They're computed once per region per task template and reused across all training epochs — this saves enormous compute.
- **Do:** Tune the spatial bias weight lambda (typically 0.1-0.4) via validation. Too high forces purely local attention; too low loses spatial coherence.
- **Do:** Use different prompt templates for different tasks even on the same city. The template is what makes the representation task-aware — generic templates defeat the purpose.
- **Do:** Start with 144-dimensional embeddings and 4 attention heads as a baseline; scale up only if underfitting.
- **Avoid:** Training the MLLM end-to-end. The frozen LLM acts as a semantic feature extractor. Fine-tuning it on small urban datasets will overfit and destroy general knowledge.
- **Avoid:** Skipping the spatial bias and relying on standard self-attention. Without B_ij, the model treats all region pairs equivalently regardless of distance, producing spatially noisy predictions.
- **Avoid:** Using a single combined loss across multiple tasks during Stage 1. Train SREL per-task or with a carefully weighted multi-task objective; naive combination degrades all tasks.

## Error Handling

| Problem | Cause | Fix |
|---------|-------|-----|
| Spatial attention collapses (all regions attend uniformly) | lambda too low or adjacency threshold too large | Increase lambda to 0.3+; tighten distance threshold so each region has 5-15 neighbors |
| Prompt vectors have near-zero variance | Prompt template too generic or MLLM not multimodal | Rewrite template with task-specific language; ensure images are actually processed |
| Stage 2 overfits quickly | Too few regions relative to alignment parameters | Reduce cross-attention heads to 2; add dropout (0.1-0.3) after alignment |
| R-squared negative or near zero | Data leakage in spatial splits or target variable not normalized | Use spatially-aware train/test splits; normalize targets to zero mean |
| MLLM API costs too high for many regions | Generating prompt vectors for thousands of grid cells | Batch API calls; use open-source models (LLaVA-7B) locally; reduce grid resolution |

## Limitations

- **Region count ceiling:** The Graphormer attention is O(N^2) in the number of regions. For cities with >5,000 regions, consider sparse attention or region clustering before applying SREL.
- **MLLM dependency:** Prompt4RE quality depends heavily on the frozen LLM's visual and geographic understanding. Models without strong spatial reasoning (e.g., text-only LLMs) will produce weak prompt vectors.
- **Data availability:** The full pipeline requires POI data, mobility records, satellite imagery, and street-view photos per region. Missing modalities degrade performance — there is no built-in imputation.
- **Urban-specific:** The spatial bias design assumes geographic proximity implies interaction. This may not hold for tasks where connectivity (e.g., transit networks) matters more than Euclidean distance.
- **Evaluation scope:** Published results cover Chicago only. Transferability to cities with different spatial scales, data quality, or urban morphology is not yet validated.

## Reference

**Paper:** [ToPT: Task-Oriented Prompt Tuning for Urban Region Representation Learning](https://arxiv.org/abs/2602.01610v1) (Guo et al., 2026). Focus on Section 3 (Methodology) for the SREL spatial bias formulation and Prompt4RE cross-attention alignment, and Section 4.3 (Ablation Studies) for evidence of each component's contribution.