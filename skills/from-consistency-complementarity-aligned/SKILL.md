---
name: "from-consistency-complementarity-aligned"
description: "Build multi-modal time series analysis pipelines that align numerical data with visual plots and textual captions using MADI's patch-level alignment and disentangled interaction framework. Use when: 'analyze this time series with both charts and numbers', 'build a multi-modal temporal reasoning system', 'align time series patches with plot images', 'combine numerical and visual time series features', 'disentangle shared vs unique signals across modalities', 'implement cross-modal time series QA'"
---

# MADI: Multi-Modal Aligned & Disentangled Time Series Understanding

This skill enables Claude to build systems that combine numerical time series data with their visual plot renderings and textual captions for joint understanding and reasoning. The core insight from the MADI framework (arXiv:2601.21436v2) is that naively concatenating multi-modal representations of time series actually *hurts* performance because of fine-grained temporal misalignment and entanglement between shared and modality-specific semantics. MADI solves this with three mechanisms: patch-level contrastive alignment, discrete vector-quantized disentanglement of shared vs. unique information, and query-conditioned critical-token highlighting. Claude applies these principles when designing pipelines that need to reason over time series using multiple representation modalities simultaneously.

## When to Use

- When the user wants to build a system that answers natural language questions about time series data by combining numerical analysis with visual chart interpretation
- When implementing a multi-modal encoder that must align time series patches (numerical segments) with corresponding regions of rendered line plots
- When the user needs to separate what's shared between numerical and visual representations of the same time series from what's unique to each modality
- When building a temporal QA pipeline that must handle both categorical questions ("Is there an upward trend?") and numerical questions ("What is the maximum value between t=50 and t=100?")
- When the user asks to improve a time series LLM that currently concatenates modalities naively and gets poor results
- When designing a vector-quantized bottleneck to enforce compact, disentangled multi-modal representations

## Key Technique

**The Problem with Naive Multi-Modal Fusion.** When you render a time series as a line plot and feed both the raw numbers and the image to an LLM, performance often *degrades* compared to using a single modality. This happens for two reasons: (1) temporal misalignment — a numerical patch covering timestamps [t_50, t_60] may not correspond to the same spatial region in the visual plot due to axis scaling, padding, and resolution differences; and (2) semantic entanglement — both modalities encode overlapping information (trend, periodicity) mixed with modality-unique information (exact values in numbers; visual shape patterns in plots), and the model wastes capacity re-encoding redundant shared features while missing complementary unique ones.

**MADI's Three-Stage Solution.** First, *Patch-level Alignment* divides the time series into non-overlapping temporal patches and renders corresponding visual patches at matching granularity, then uses InfoNCE contrastive loss to align numerical patch embeddings with their visual and textual counterparts — critically, gradients only flow through the numerical encoder while visual/textual encoders are frozen via stop-gradient. Second, *Discrete Disentangled Interaction* uses hierarchical Residual Vector Quantization (RVQ) with M codebook levels of increasing resolution to compress each modality's embeddings into discrete latent codes representing the *shared* semantics; the *unique* information is extracted as the residual (original embedding minus quantized common code), and cross-attention then fuses only the unique components across modalities, preventing redundant shared information from interfering. Third, *Critical-token Highlighting* uses two parallel cross-attention branches — one conditioned on the user's question, one using learned modality-intrinsic queries — to select and prepend the most informative tokens before feeding into the LLM backbone.

**Why Discretization Matters.** Continuous disentanglement (e.g., via simple projection heads) allows redundant or noisy correlations to leak between the shared and unique components. Vector quantization forces shared semantics into a compact codebook, creating a hard information bottleneck that produces cleaner separation. The hierarchical RVQ adds multi-resolution structure: coarse codebooks capture global trends while fine codebooks capture local fluctuations.

## Step-by-Step Workflow

1. **Segment time series into patches.** Divide each input time series of length T into non-overlapping patches of size p_n, producing T̃ = T / p_n patches. Each patch is a contiguous temporal window (e.g., 16 or 32 timesteps). Store timestamps with each patch for physical grounding.

2. **Render aligned visual patches.** Plot the full time series as a line chart at resolution (T̃ · p_v) × p_v, where p_v is the visual patch size. Decompose the rendered image into T̃ visual patches, each spatially corresponding to one numerical patch. This physical grounding ensures patch j in both modalities covers the same temporal window.

3. **Generate structured textual captions per patch.** For each patch, produce a caption containing: timestamp range, max, min, mean, standard deviation, and qualitative trend description. Format as structured text (e.g., `"[t=48..64] max=3.21, min=-0.54, mean=1.12, std=0.89, trend=rising"`).

4. **Encode each modality independently.** Encode numerical patches through a learnable linear projection + positional embeddings + transformer blocks → E_n ∈ R^{T̃×D}. Encode visual patches through a frozen pre-trained vision encoder (e.g., SigLIP from Qwen2.5-VL) → E_v ∈ R^{T̃×D}. Encode textual captions through the LLM tokenizer + mean pooling → E_s ∈ R^{T̃×D}.

5. **Apply patch-level contrastive alignment.** Compute InfoNCE loss using numerical embeddings as anchors and visual/textual embeddings as positives. Apply stop-gradient on the visual and textual sides so only the numerical encoder learns to align. Use temperature τ ≈ 0.07. Loss: L_pa = L_align_nv + L_align_ns.

6. **Disentangle shared vs. unique via hierarchical RVQ.** Project aligned embeddings to lower dimension d. Apply M levels of Residual Vector Quantization: level m has K·2^{m-1} codes at temporal resolution 2^{M-m}. At each level, pool residuals into segments, quantize via nearest-codebook lookup, compute new residual. Sum all quantized levels → shared representation Z. Compute unique as residual: U_n = E_n - Z_n, U_v = E_v - Z_v.

7. **Enforce disentanglement with regularization losses.** Apply bidirectional contrastive loss between Z_n and Z_v to ensure shared codes capture the same semantics across modalities. Apply orthogonality loss between Z and U within each modality: L_orth = mean(|sim(z_j, u_j)|), pushing shared and unique representations to encode non-overlapping information.

8. **Fuse unique information via cross-attention.** Compute Ē_n = E_n + CrossAttn(E_n, U_v, U_v) and Ē_v = E_v + CrossAttn(E_v, U_n, U_n). This enriches each modality with only the complementary information from the other, without redundant shared signals.

9. **Highlight critical tokens conditioned on the query.** Use H learnable query vectors with two parallel cross-attention branches: (a) question-conditioned — compress question embeddings into queries, attend over modality tokens; (b) modality-intrinsic — learned queries attend over modality tokens to find salient patterns. Sum both outputs and prepend as H special tokens before the sequence.

10. **Assemble final input and generate.** Concatenate: [system_context, question_tokens, critical_tokens_n, patch_tokens_n, critical_tokens_v, patch_tokens_v] → feed to LLM backbone. Train with combined loss: L = L_lm + λ_1·L_pa + λ_2·L_DDI, where L_DDI = L_vq + α·L_com + β·L_orth.

## Concrete Examples

**Example 1: Building a multi-modal time series QA system**

User: "I have sensor time series data and I want to build a system that can answer questions like 'When did the temperature spike?' by looking at both the raw numbers and rendered plots."

Approach:
1. Patch the sensor data into 32-timestep windows
2. Render each time series as a matplotlib line plot at matched resolution
3. Generate per-patch statistical captions automatically
4. Build a dual-encoder: numerical branch (linear + 4-layer transformer), visual branch (frozen CLIP/SigLIP)
5. Align with patch-level InfoNCE (stop-grad on visual side)
6. Add 3-level RVQ with codebook sizes [64, 128, 256] to disentangle shared/unique
7. Implement cross-attention fusion on unique residuals only
8. Add query-conditioned token highlighting with H=8 learned queries
9. Feed into a 7B instruction-tuned LLM for answer generation

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class PatchAligner(nn.Module):
    """Patch-level contrastive alignment between numerical and visual encodings."""
    def __init__(self, dim=512, temperature=0.07):
        super().__init__()
        self.temperature = temperature
        self.num_encoder = nn.Sequential(
            nn.Linear(32, dim),  # patch_size -> dim
            nn.TransformerEncoder(
                nn.TransformerEncoderLayer(d_model=dim, nhead=8, batch_first=True),
                num_layers=4
            )
        )
        # Visual encoder is frozen; no parameters here

    def forward(self, num_patches, vis_embeddings):
        # num_patches: (B, T̃, patch_size), vis_embeddings: (B, T̃, D)
        e_n = self.num_encoder(num_patches)  # (B, T̃, D)
        # InfoNCE with stop-gradient on visual side
        e_v = vis_embeddings.detach()  # stop-gradient
        sim = torch.einsum('bid,bjd->bij',
                           F.normalize(e_n, dim=-1),
                           F.normalize(e_v, dim=-1)) / self.temperature
        labels = torch.arange(sim.size(1), device=sim.device)
        loss = F.cross_entropy(sim.flatten(0, 1),
                               labels.unsqueeze(0).expand(sim.size(0), -1).flatten())
        return e_n, loss


class HierarchicalRVQ(nn.Module):
    """Residual Vector Quantization for disentangling shared semantics."""
    def __init__(self, dim=512, latent_dim=64, M=3, K_base=64):
        super().__init__()
        self.M = M
        self.proj_down = nn.Linear(dim, latent_dim)
        self.proj_up = nn.Linear(latent_dim, dim)
        self.codebooks = nn.ParameterList([
            nn.Parameter(torch.randn(K_base * (2 ** m), latent_dim))
            for m in range(M)
        ])

    def quantize_level(self, residual, codebook, window_size):
        B, T, d = residual.shape
        # Pool into segments of window_size
        n_segs = T // window_size
        pooled = residual[:, :n_segs * window_size].reshape(B, n_segs, window_size, d).mean(dim=2)
        # Nearest codebook lookup
        dists = torch.cdist(pooled, codebook.unsqueeze(0))
        indices = dists.argmin(dim=-1)
        quantized_segs = codebook[indices]  # (B, n_segs, d)
        # Expand back to full resolution
        quantized = quantized_segs.unsqueeze(2).expand(-1, -1, window_size, -1)
        quantized = quantized.reshape(B, n_segs * window_size, d)
        # Pad if needed
        if n_segs * window_size < T:
            quantized = F.pad(quantized, (0, 0, 0, T - n_segs * window_size))
        return quantized, residual - quantized

    def forward(self, embeddings):
        z = self.proj_down(embeddings)
        residual = z
        total_quantized = torch.zeros_like(z)
        for m in range(self.M):
            window_size = 2 ** (self.M - 1 - m)
            q, residual = self.quantize_level(residual, self.codebooks[m], max(window_size, 1))
            total_quantized = total_quantized + q
        # Straight-through estimator
        quantized_st = z + (total_quantized - z).detach()
        shared = self.proj_up(quantized_st)
        unique = embeddings - shared
        # Commitment loss
        vq_loss = F.mse_loss(total_quantized.detach(), z) + F.mse_loss(total_quantized, z.detach())
        return shared, unique, vq_loss
```

**Example 2: Improving an existing time series + chart LLM**

User: "My current model concatenates time series embeddings and chart image embeddings before feeding them to an LLM, but performance is worse than using either modality alone. How do I fix this?"

Approach:
1. Diagnose: naive concatenation causes entanglement — the model re-encodes redundant trend/shape info from both modalities while missing modality-unique signals (exact numerical values, visual pattern gestalts)
2. Add patch-level alignment to ensure the two modalities' representations are temporally registered (patch j in numbers = patch j in the chart)
3. Insert a discrete bottleneck (RVQ) between the encoders and the LLM to split each modality into shared codes + unique residuals
4. Replace naive concatenation with unique-centric cross-attention: each modality absorbs only the *complementary* information from the other
5. Add orthogonality regularization to prevent information leakage between shared and unique components

```
Before (naive):  E_combined = Concat(E_num, E_vis) → LLM   ❌ Entangled
After (MADI):    Z_shared = RVQ(E_num) ≈ RVQ(E_vis)         # aligned shared codes
                 U_num = E_num - Z_num                        # unique to numbers
                 U_vis = E_vis - Z_vis                        # unique to visuals
                 Ē_num = E_num + CrossAttn(E_num, U_vis)      # enrich with complement
                 Ē_vis = E_vis + CrossAttn(E_vis, U_num)
                 → [critical_tokens, Ē_num, Ē_vis] → LLM    ✅ Disentangled
```

**Example 3: Implementing query-conditioned critical-token highlighting**

User: "I have a time series QA model but it struggles with long sequences because most patches are irrelevant to the question. How can I focus the model on the relevant temporal region?"

Approach:
1. Implement dual-branch critical-token selection
2. Branch A: encode the question, compress into H=8 learned query vectors via cross-attention, then attend over all time series tokens → question-relevant highlights
3. Branch B: use H=8 learned modality-intrinsic queries that attend over tokens to find inherently salient patterns (spikes, regime changes) regardless of the question
4. Sum both branches, prepend as special prefix tokens before the full sequence

```python
class CriticalTokenHighlighter(nn.Module):
    def __init__(self, dim=512, n_queries=8):
        super().__init__()
        self.query_conditioned = nn.Parameter(torch.randn(n_queries, dim))
        self.modality_intrinsic = nn.Parameter(torch.randn(n_queries, dim))
        self.q_cross_attn = nn.MultiheadAttention(dim, num_heads=8, batch_first=True)
        self.m_cross_attn = nn.MultiheadAttention(dim, num_heads=8, batch_first=True)
        self.question_compressor = nn.MultiheadAttention(dim, num_heads=8, batch_first=True)

    def forward(self, modality_tokens, question_embeddings):
        B = modality_tokens.size(0)
        # Question-conditioned branch
        q_queries = self.query_conditioned.unsqueeze(0).expand(B, -1, -1)
        q_compressed, _ = self.question_compressor(q_queries, question_embeddings, question_embeddings)
        h_question, _ = self.q_cross_attn(q_compressed, modality_tokens, modality_tokens)
        # Modality-intrinsic branch
        m_queries = self.modality_intrinsic.unsqueeze(0).expand(B, -1, -1)
        h_intrinsic, _ = self.m_cross_attn(m_queries, modality_tokens, modality_tokens)
        # Combine and prepend
        critical = h_question + h_intrinsic  # (B, H, D)
        return torch.cat([critical, modality_tokens], dim=1)  # (B, H+T̃, D)
```

## Best Practices

- **Do:** Ensure physical temporal correspondence when creating visual patches — the x-axis range of visual patch j must exactly match the timestamp range of numerical patch j. Misaligned patches defeat the purpose of patch-level alignment.
- **Do:** Use stop-gradient on the visual and textual encoder during contrastive alignment. The goal is to pull the numerical representations toward the pre-trained visual/textual feature space, not distort the frozen encoders.
- **Do:** Start with M=3 RVQ levels and K_base=64 codebook entries. Increase codebook size if reconstruction is poor (high VQ loss); decrease if codebook utilization drops below 50%.
- **Do:** Monitor codebook utilization — dead codes indicate the bottleneck is too wide. Apply codebook reset or EMA updates for underused entries.
- **Avoid:** Skipping the orthogonality loss between shared and unique components. Without it, shared semantics leak into the unique residuals, defeating disentanglement.
- **Avoid:** Applying gradients through the visual encoder's contrastive loss. This corrupts the pre-trained visual features that serve as alignment targets.
- **Avoid:** Using critical-token highlighting with very small H (e.g., H=1). The dual-branch design needs enough query capacity to capture both question-relevant and intrinsically salient patterns. H=8 is a good default.

## Error Handling

- **Codebook collapse (all inputs map to same few codes):** Monitor per-code usage frequency. If utilization drops below 30%, apply exponential moving average codebook updates or periodic reinitialization of dead codes from encoder outputs.
- **Alignment loss diverging:** Check that visual and numerical patches are temporally registered. Verify the contrastive temperature τ is not too low (causes gradient spikes) or too high (loss plateaus). Start with τ=0.07.
- **Unique residuals are near-zero:** The orthogonality weight β may be too high, forcing unique components to collapse. Reduce β or verify that the shared codebook isn't too large (overfitting all information into shared codes).
- **Question-conditioned highlighting attends uniformly:** The question encoder may not be providing discriminative enough representations. Ensure question embeddings are from a reasonably deep layer of the LLM, not just token embeddings.
- **Visual patch resolution mismatch:** If the rendered plot resolution doesn't divide evenly into T̃ patches, artifacts at patch boundaries will hurt alignment. Always render at exactly T̃ · p_v pixel width.

## Limitations

- Requires rendering time series as plots at training time, adding I/O and compute overhead proportional to dataset size. Not suitable for ultra-low-latency inference without pre-cached renderings.
- The RVQ codebook must be sized appropriately for the data distribution; there is no automatic codebook sizing, and poor choices lead to either collapse or underutilization.
- Assumes univariate or low-dimensional multivariate time series that can be meaningfully visualized as line plots. High-dimensional multivariate data (e.g., 100+ channels) cannot be effectively rendered into a single interpretable plot.
- The patch-level alignment assumes a linear mapping between temporal index and spatial position in the plot. Non-standard visualizations (log-scale axes, broken axes, multi-panel plots) break this assumption.
- Performance gains are strongest for mixed categorical+numerical QA tasks. For purely numerical extraction tasks (e.g., "what is the value at t=57?"), a well-tuned numerical-only model may suffice.

## Reference

**Paper:** Ni, H., Zhang, W., Wang, F., Shao, Z., & Liu, H. (2026). *From Consistency to Complementarity: Aligned and Disentangled Multi-modal Learning for Time Series Understanding and Reasoning.* arXiv:2601.21436v2. [https://arxiv.org/abs/2601.21436v2](https://arxiv.org/abs/2601.21436v2)

**Key takeaway:** Look at Section 3 for the full MADI architecture — particularly the hierarchical RVQ formulation (Eq. 5-9) for discrete disentanglement, and Section 4.3 for ablations showing that removing any of the three components (PA, DDI, CTH) consistently degrades performance, with DDI providing the largest individual contribution.