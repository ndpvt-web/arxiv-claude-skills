---
name: "explicit-multi-head-attention-inter-head"
description: |
  Implement Multi-head Explicit Attention (MEA) with inter-head interaction for Transformer models.
  Adds Head-level Linear Composition (HLC) modules and head-level Group Normalization to standard
  multi-head attention, enabling cross-head communication, faster convergence with larger learning
  rates, and 50% KV-cache compression via virtual heads.

  Trigger phrases:
  - "Add inter-head interaction to my attention layer"
  - "Implement MEA attention with head-level linear composition"
  - "Compress KV-cache using virtual heads"
  - "Replace multi-head attention with explicit cross-head mixing"
  - "Add head-level normalization to my Transformer"
  - "Reduce KV-cache memory with low-rank head reconstruction"
---

# Explicit Multi-head Attention with Inter-head Interaction (MEA)

This skill enables Claude to implement Multi-head Explicit Attention (MEA), an attention variant from Peng et al. (2026) that explicitly models cross-head interaction in Transformer models. MEA introduces two components on top of standard multi-head attention: (1) a Head-level Linear Composition (HLC) module that applies learnable linear combinations to key and value vectors across heads, and (2) head-level RMSNorm that stabilizes the recombined representations. The technique improves pretraining robustness, allows larger learning rates for faster convergence, and enables a practical KV-cache compression strategy that halves memory usage with minimal accuracy loss.

## When to Use

- When the user is building or modifying a Transformer model and wants attention heads to share information rather than operate independently
- When implementing a custom attention layer in PyTorch/JAX and the user asks for "inter-head interaction" or "cross-head mixing"
- When the user needs to reduce KV-cache memory during LLM inference without retraining from scratch
- When pretraining a language model and encountering instability at higher learning rates
- When the user wants to compress a multi-head attention model by replacing physical heads with fewer "virtual heads" reconstructed via linear combination
- When adapting a pretrained model (continued pretraining) and the user wants to inject MEA layers with SVD-based initialization

## Key Technique

**Head-level Linear Composition (HLC):** Standard multi-head attention projects inputs into h independent heads that never communicate. HLC adds a learnable weight matrix `W_lc ∈ R^{h' x h}` that linearly recombines heads before attention computation. Concretely, given component key tensors `K_comp ∈ R^{n x h' x d}` (n = sequence length, h' = component heads, d = head dimension), HLC produces composite keys via the einsum: `K_lc = einsum("n h' d, h' h -> n h d", K_comp, W_lc^K)`. The same operation is applied separately to values with its own matrix `W_lc^V`. This is cheap -- only `2 * h' * h` additional parameters per layer -- but it allows every composite head to be an arbitrary linear mix of all component heads, enabling rich inter-head communication.

**Head-level RMSNorm:** After HLC recombines the heads, their statistical properties can diverge, destabilizing training. MEA applies RMSNorm across the head dimension to the concatenated output before the final projection. This normalization preserves representational diversity while preventing gradient explosion, which is why MEA tolerates learning rates up to 3x larger than standard MHA (e.g., 3e-3 vs 1e-3).

**Virtual Heads for KV-Cache Compression:** For inference efficiency, MEA decomposes the key/value projection matrices via SVD: `W^K ≈ W̃^{K'} ⊗ W̃_lc^K`, where `W̃^{K'}` projects to h' < h component heads and `W̃_lc^K` reconstructs h composite heads. During inference, only the h' component KV pairs are cached. With h' = h/2, this cuts KV-cache memory by 50% with negligible loss on knowledge and reasoning tasks, and only ~3.6% accuracy drop on Olympiad-level math.

## Step-by-Step Workflow

1. **Identify the target attention module.** Locate the standard `MultiHeadAttention` class in the codebase. Identify the number of heads `h`, head dimension `d_k` (and `d_v`), and how Q/K/V projections are structured (typically `nn.Linear(d_model, h * d_k)`).

2. **Add HLC weight matrices.** Create two learnable parameters: `W_lc_K` and `W_lc_V`, each of shape `(h_component, h_composite)`. For full MEA (no compression), set `h_component = h_composite = h`. Initialize them as identity matrices so the model starts equivalent to standard MHA.

3. **Implement the HLC forward pass.** After computing K and V tensors and reshaping to `(batch, h_component, seq_len, d_k)`, apply the linear combination:
   ```python
   # K shape: (B, h', N, d_k), W_lc_K shape: (h', h)
   K_lc = torch.einsum("b c n d, c h -> b h n d", K, self.W_lc_K)
   V_lc = torch.einsum("b c n d, c h -> b h n d", V, self.W_lc_V)
   ```
   Use `K_lc` and `V_lc` in place of K and V for the standard scaled dot-product attention with Q.

4. **Add head-level RMSNorm.** After computing attention output `O ∈ (B, h, N, d_v)`, reshape to `(B, N, h * d_v)` and apply RMSNorm (or GroupNorm with `num_groups=h`) before the output projection. This stabilizes the recombined head representations.

5. **Verify correctness with identity initialization.** Run a forward pass and confirm the output matches standard MHA exactly when `W_lc_K` and `W_lc_V` are identity matrices. This is your regression test.

6. **Configure training hyperparameters.** MEA tolerates larger learning rates. Start with 2-3x your baseline peak LR (e.g., 3e-3 instead of 1e-3) with AdamW (weight decay 0.1) and cosine annealing decaying to 10% of peak.

7. **[Optional] Enable virtual head compression for inference.** To compress KV-cache, reduce `h_component` to `h' < h` (e.g., `h' = h/2`). Compute SVD of the pretrained K/V projection weights: `W^K = U @ diag(S) @ Vt`. Set `W̃^{K'} = U[:, :h'*d_k] @ diag(S[:h'*d_k])` as the new K projection (producing h' heads) and `W̃_lc^K = Vt[:h'*d_k, :].reshape(h', h)` as the HLC matrix.

8. **[Optional] Fine-tune after compression.** After SVD initialization, run continued pretraining for a short recovery phase (e.g., 1-5B tokens) to let the model adapt to the compressed representation. This recovers most of the accuracy gap.

9. **Validate on downstream tasks.** Evaluate on a representative benchmark suite (e.g., PIQA, HellaSwag, ARC, WinoGrande) to confirm MEA matches or exceeds baseline MHA performance.

10. **Profile memory and latency.** Measure KV-cache memory usage during inference with and without virtual head compression. Confirm the expected ~50% reduction when using h' = h/2.

## Concrete Examples

**Example 1: Adding MEA to a GPT-style model in PyTorch**

User: "I have a standard multi-head attention implementation. Add inter-head interaction using the MEA technique."

Approach:
1. Read the existing attention class to identify head count and dimensions
2. Add HLC parameters and modify the forward pass
3. Add head-level normalization

Output (key modifications to the attention class):
```python
class MEAAttention(nn.Module):
    def __init__(self, d_model, n_heads, n_component_heads=None):
        super().__init__()
        self.n_heads = n_heads
        self.n_comp = n_component_heads or n_heads  # h' component heads
        self.d_k = d_model // n_heads

        # Standard Q/K/V projections (K/V project to component heads)
        self.W_q = nn.Linear(d_model, n_heads * self.d_k, bias=False)
        self.W_k = nn.Linear(d_model, self.n_comp * self.d_k, bias=False)
        self.W_v = nn.Linear(d_model, self.n_comp * self.d_k, bias=False)
        self.W_o = nn.Linear(d_model, d_model, bias=False)

        # HLC: learnable cross-head mixing matrices
        self.W_lc_K = nn.Parameter(torch.eye(self.n_comp, n_heads))
        self.W_lc_V = nn.Parameter(torch.eye(self.n_comp, n_heads))

        # Head-level RMSNorm
        self.head_norm = nn.RMSNorm(self.d_k)

    def forward(self, x, mask=None):
        B, N, _ = x.shape
        Q = self.W_q(x).view(B, N, self.n_heads, self.d_k).transpose(1, 2)
        K = self.W_k(x).view(B, N, self.n_comp, self.d_k).transpose(1, 2)
        V = self.W_v(x).view(B, N, self.n_comp, self.d_k).transpose(1, 2)

        # HLC: mix component heads into composite heads
        K = torch.einsum("b c n d, c h -> b h n d", K, self.W_lc_K)
        V = torch.einsum("b c n d, c h -> b h n d", V, self.W_lc_V)

        # Standard scaled dot-product attention
        attn = (Q @ K.transpose(-2, -1)) / (self.d_k ** 0.5)
        if mask is not None:
            attn = attn.masked_fill(mask == 0, float("-inf"))
        attn = torch.softmax(attn, dim=-1)
        O = attn @ V  # (B, h, N, d_k)

        # Head-level RMSNorm before output projection
        O = self.head_norm(O)
        O = O.transpose(1, 2).reshape(B, N, -1)
        return self.W_o(O)
```

**Example 2: Compressing KV-cache of a pretrained model via SVD**

User: "I have a pretrained 7B model with 32 heads. Compress its KV-cache by 50% using virtual heads."

Approach:
1. Extract K/V projection weights from each layer
2. Perform SVD and split into component projection + HLC matrix
3. Replace attention modules with MEA variants
4. Run short recovery fine-tuning

Output (SVD compression script):
```python
import torch

def compress_kv_projections(model, target_component_heads):
    """Replace K/V projections with low-rank virtual head equivalents."""
    for layer in model.transformer.layers:
        attn = layer.attention
        h = attn.n_heads
        h_prime = target_component_heads  # e.g., h // 2
        d_k = attn.d_k

        for proj_name, lc_name in [("W_k", "W_lc_K"), ("W_v", "W_lc_V")]:
            W = getattr(attn, proj_name).weight.data  # (h*d_k, d_model)
            U, S, Vt = torch.linalg.svd(W, full_matrices=False)

            # Keep top h'*d_k singular components
            rank = h_prime * d_k
            W_comp = U[:, :rank] @ torch.diag(S[:rank])  # new projection
            W_lc = Vt[:rank, :].reshape(h_prime, d_k, -1)  # reshaped for HLC

            # Replace projection to produce h' component heads
            new_proj = torch.nn.Linear(W.shape[1], rank, bias=False)
            new_proj.weight.data = W_comp
            setattr(attn, proj_name, new_proj)

            # Set HLC matrix (h' x h) from SVD factors
            # Simplified: compute mixing weights from SVD structure
            lc_matrix = torch.eye(h_prime, h)  # initialize, refine during recovery
            setattr(attn, lc_name, torch.nn.Parameter(lc_matrix))

        attn.n_comp = h_prime
    return model
```

**Example 3: Stabilizing pretraining with higher learning rates**

User: "My 1.3B Transformer diverges when I increase the learning rate above 1e-3. How can MEA help?"

Approach:
1. Replace standard MHA with MEA (identity-initialized HLC + head RMSNorm)
2. The head-level RMSNorm is the key stabilizer -- it prevents gradient explosion from head recombination
3. Increase peak LR to 3e-3 with cosine annealing to 3e-4

Output (training config adjustment):
```python
# Before: standard MHA, limited to lr=1e-3
# After: MEA with head-level RMSNorm enables 3x larger LR

optimizer = torch.optim.AdamW(model.parameters(), lr=3e-3, weight_decay=0.1)
scheduler = torch.optim.lr_scheduler.CosineAnnealingLR(
    optimizer, T_max=total_steps, eta_min=3e-4  # decay to 10% of peak
)
# MEA's head-level normalization prevents the divergence that standard MHA
# encounters at this learning rate, leading to faster convergence and
# lower final validation loss.
```

## Best Practices

- **Do:** Initialize `W_lc_K` and `W_lc_V` as identity matrices when training from scratch or inserting MEA into an existing architecture. This ensures the model starts equivalent to standard MHA and learns inter-head interaction gradually.
- **Do:** Apply RMSNorm per-head (treating each head's d_k features as the normalization dimension), not across all heads concatenated. This preserves each head's representational identity while stabilizing scale.
- **Do:** When compressing via virtual heads, use SVD of the pretrained projection matrices as initialization rather than random initialization. This preserves most of the pretrained knowledge.
- **Do:** Keep the HLC matrices relatively small -- they are `(h' x h)` scalars, not full feature transformations. This is intentionally lightweight.
- **Avoid:** Applying HLC to the query projections. The paper applies it only to keys and values, keeping queries independent. Mixing queries introduces unnecessary coupling.
- **Avoid:** Aggressive compression ratios beyond 50% (h' < h/2) without extensive recovery fine-tuning. The paper shows diminishing returns and sharper accuracy drops at higher compression.

## Error Handling

- **Shape mismatch after HLC:** If `n_component_heads != n_heads`, ensure Q still has `n_heads` heads while K/V have `n_component_heads` before HLC. After HLC, K/V should match Q's head count. Validate shapes with assertions: `assert K_lc.shape[1] == Q.shape[1]`.
- **Training divergence despite MEA:** If loss spikes even with MEA, verify that head-level RMSNorm is applied *before* the output projection, not after. Also check that `W_lc` gradients are not exploding -- add gradient clipping (max_norm=1.0) as a safeguard.
- **SVD compression produces NaN:** This can happen if projection matrices have very small singular values. Clamp singular values to a minimum threshold (e.g., 1e-6) before constructing the compressed weights.
- **KV-cache not actually smaller:** Ensure the inference engine is caching the component heads (h' heads of dimension d_k) and applying HLC on-the-fly during attention, not caching the full h composite heads after HLC expansion.

## Limitations

- MEA adds a small computational overhead per layer (the einsum for HLC). For very latency-sensitive serving, profile to confirm the overhead is acceptable -- it is typically negligible compared to the attention computation itself but adds up across layers.
- The virtual head KV-cache compression works best for knowledge retrieval and scientific reasoning. Olympiad-level mathematical reasoning shows a ~3.6% accuracy drop at 50% compression, suggesting that math-heavy tasks rely on the full head capacity more than other tasks.
- The technique is designed for decoder-only and encoder-decoder Transformers. Applying it to non-attention architectures (e.g., state-space models) requires rethinking the HLC concept.
- SVD-based compression initialization assumes the pretrained projection matrices are well-conditioned. Models with poorly trained or undertrained layers may not compress well.
- The paper validates on models up to ~7B parameters. Scaling behavior to 70B+ models is plausible but not empirically confirmed in the paper.

## Reference

**Paper:** [Explicit Multi-head Attention for Inter-head Interaction in Large Language Models](https://arxiv.org/abs/2601.19611v1) (Peng et al., 2026). Look for Section 3 (MEA formulation and HLC definition), Section 4 (virtual head compression via SVD), and Tables 1-3 (benchmark comparisons showing MEA advantages at higher learning rates and with KV-cache compression).