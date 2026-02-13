---
name: "c2rope-causal-continuous-rotary-positional"
description: |
  Implement C²RoPE (Causal Continuous Rotary Positional Encoding) for multimodal transformers
  that process 2D/3D visual data alongside text. Replaces standard 1D RoPE with a triplet
  (m, x, y) positional index and Chebyshev causal masking to preserve spatial locality in
  vision-language models.

  Trigger phrases:
  - "implement C2RoPE positional encoding"
  - "fix spatial locality loss in vision-language RoPE"
  - "add 2D-aware rotary embeddings for image tokens"
  - "implement Chebyshev causal masking for visual attention"
  - "modify RoPE for multimodal 3D reasoning"
  - "spatially-aware positional encoding for multi-view images"
---

# C²RoPE: Causal Continuous Rotary Positional Encoding for Multimodal Transformers

This skill enables Claude to implement C²RoPE, a drop-in replacement for standard RoPE in vision-language models that fixes two fundamental problems: (1) spatial locality loss caused by flattening 2D image patches into 1D sequences, and (2) long-term attention decay that causes models to neglect earlier visual tokens as sequence length grows. C²RoPE achieves this by constructing a triplet positional index (temporal, x, y) with a frequency allocation strategy, and introducing Chebyshev distance-based causal masking for visual self-attention.

## When to Use

- When building or modifying a multimodal transformer (e.g., LLaVA, Qwen-VL) that processes image tokens alongside text and you need to preserve spatial relationships between image patches
- When visual question answering performance degrades on tasks requiring spatial reasoning, such as "what is to the left of X" or "describe the layout of the scene"
- When implementing multi-view 3D scene understanding where multiple images are flattened into a single sequence and the model loses track of spatial structure
- When attention maps show that early visual tokens receive negligible attention weight due to standard RoPE's temporal decay bias
- When adapting a pretrained LLM (LLaMA, Mistral) for vision tasks and want to modify its positional encoding without retraining from scratch
- When implementing custom attention masks that respect 2D spatial causality rather than 1D sequential causality

## Key Technique

**The core problem:** Standard RoPE assigns each token a single integer position index m = 0, 1, 2, ..., then computes rotation matrices R(m) that cause attention to decay with distance |m_q - m_k|. When a 24x24 image grid is flattened row-by-row, two vertically adjacent patches (e.g., positions 0 and 24) receive distant indices, breaking spatial continuity along the column dimension. Additionally, image tokens placed early in the sequence receive progressively less attention as text tokens extend the sequence -- a temporal decay bias inherited from language modeling that is inappropriate for visual content.

**C²RoPE's solution has two components.** First, it replaces the scalar position index m with a triplet (m, x, y) where m is the original temporal index and (x, y) are Cartesian coordinates centered at the image. For a sqrt(v) x sqrt(v) image patch grid, coordinates range from (1 - sqrt(v)/2) to (sqrt(v)/2 - 1) along each axis. The embedding dimension d is then split: the first (d - d_spatial) dimensions encode temporal position m using standard RoPE frequencies theta_i = 10000^(-2(i-1)/d), while the remaining d_spatial dimensions interleave x and y coordinates. In the paper's configuration, d=128 uses 96 dimensions for temporal and 32 for spatial. The spatial dimensions use higher-frequency slots (lower dimension indices within their allocation) so they capture fine-grained spatial distinctions without disrupting the LLM's pretrained temporal position understanding in the lower frequencies.

**Second, Chebyshev Causal Masking** replaces the standard causal (lower-triangular) attention mask for visual self-attention. Instead of enforcing that token i can only attend to tokens j <= i (which is arbitrary for 2D image patches), C²RoPE computes the Chebyshev distance from the image center for each token: d_cheb = max(|x|, |y|). Tokens at the same Chebyshev distance form a "ring" around the center and are treated as causally equivalent -- they can attend to each other freely. Tokens can attend to any token with equal or smaller Chebyshev distance (closer to center), but not to tokens with larger distance. This creates concentric square rings of causality emanating from the image center, which is a natural prior for visual processing where context flows from global (center) to peripheral regions.

## Step-by-Step Workflow

1. **Identify the existing RoPE implementation** in your model codebase. In HuggingFace Transformers-based models, this is typically in `modeling_llama.py` or equivalent, in the `LlamaRotaryEmbedding` class and the `rotate_half` / `apply_rotary_pos_emb` functions. Note the embedding dimension `d` (typically 128 per head).

2. **Define the triplet position index builder.** For each image in the input, compute the spatial grid dimensions (e.g., 24x24 = 576 patches). Assign each patch coordinates (x, y) centered at the image: `x = col - (W-1)/2`, `y = (H-1)/2 - row` where row, col are the patch's grid position. Retain the original temporal index m from the sequence position.

3. **Implement the frequency allocation split.** Partition the RoPE dimension d into temporal dimensions d_t and spatial dimensions d_s (paper uses d_t=96, d_s=32 for d=128). For temporal dimensions, compute frequencies as standard: `theta_i = base^(-2i/d)` for i in 0..d_t/2. For spatial dimensions, interleave x and y: odd spatial slots encode x, even encode y, using frequencies from the corresponding dimension indices.

4. **Build the extended rotation matrix.** For each token position, construct the rotation angles as:
   ```python
   # Temporal component (first d_t dimensions)
   angles_t = m * theta[0:d_t//2]  # shape: (seq_len, d_t//2)
   # Spatial component (last d_s dimensions, interleaved x/y)
   angles_x = x * theta[d_t//2::2]  # even slots of spatial portion
   angles_y = y * theta[d_t//2+1::2]  # odd slots of spatial portion
   angles_s = interleave(angles_x, angles_y)  # shape: (seq_len, d_s//2)
   angles = concat(angles_t, angles_s)  # shape: (seq_len, d//2)
   ```

5. **Apply the rotation to queries and keys** using the standard RoPE mechanism (cos/sin rotation), but now with the extended angle tensor from step 4. For text tokens, set x=0 and y=0 so the spatial component contributes nothing and behavior matches standard RoPE exactly.

6. **Implement Chebyshev Causal Masking.** For each image's visual tokens, compute `d_cheb[i] = max(|x_i|, |y_i|)` for every token i. Build the attention mask where `mask[i][j] = 1 if d_cheb[j] <= d_cheb[i], else 0`. Tokens at equal Chebyshev distance can attend to each other (both directions). This mask only applies to visual-to-visual attention; text-to-text and cross-modal attention retain standard causal masking.

7. **Handle multi-image inputs.** When multiple images appear in one sequence (e.g., multi-view 3D), reset the (x, y) coordinate system for each image independently. The temporal index m continues incrementing across images. Each image gets its own Chebyshev mask block; cross-image attention uses standard causal masking.

8. **Integrate with the forward pass.** Modify the model's attention module to accept the extended position IDs (shape: batch x seq_len x 3 instead of batch x seq_len) and route them through your modified RoPE. Inject the Chebyshev mask into the attention score computation alongside any existing masks.

9. **Validate with a diagnostic test.** Feed an image with a known spatial pattern (e.g., a checkerboard) and verify that (a) attention weights between vertically adjacent patches are comparable to horizontally adjacent ones (spatial continuity restored), and (b) center patches receive attention from peripheral ones but not vice-versa (Chebyshev causality).

10. **Fine-tune or evaluate.** The modification is compatible with pretrained LLM weights since text tokens behave identically to standard RoPE. Image encoder weights may benefit from a short fine-tuning phase to adapt to the new positional structure.

## Concrete Examples

**Example 1: Adding C²RoPE to a LLaVA-style model**

User: "I'm building a LLaVA variant for 3D scene QA. Images get flattened to 576 tokens each but the model struggles with spatial questions like 'what is behind the chair'. Help me implement C²RoPE."

Approach:
1. Locate the RoPE application in the LLaMA backbone (e.g., `LlamaAttention.forward`)
2. Create a position ID builder that tags each visual token with (m, x, y):

```python
def build_c2rope_position_ids(input_ids, image_token_id, patch_grid_h=24, patch_grid_w=24):
    """Build triplet position IDs for C²RoPE."""
    batch_size, seq_len = input_ids.shape
    # Default: temporal-only positions for text tokens
    pos_ids = torch.zeros(batch_size, seq_len, 3, dtype=torch.float)
    pos_ids[:, :, 0] = torch.arange(seq_len).unsqueeze(0)  # temporal m

    for b in range(batch_size):
        img_mask = (input_ids[b] == image_token_id)
        img_starts = torch.where(img_mask)[0]
        if len(img_starts) == 0:
            continue
        # Group consecutive image tokens into images
        splits = torch.where(torch.diff(img_starts) > 1)[0] + 1
        image_groups = torch.tensor_split(img_starts, splits.tolist())

        for group in image_groups:
            n_patches = len(group)
            h, w = patch_grid_h, patch_grid_w
            for idx, seq_pos in enumerate(group):
                row, col = idx // w, idx % w
                x = col - (w - 1) / 2.0
                y = (h - 1) / 2.0 - row
                pos_ids[b, seq_pos, 1] = x  # spatial x
                pos_ids[b, seq_pos, 2] = y  # spatial y
    return pos_ids
```

3. Modify the rotary embedding to split frequencies:

```python
class C2RotaryEmbedding(nn.Module):
    def __init__(self, dim, base=10000, d_temporal=96, d_spatial=32):
        super().__init__()
        assert d_temporal + d_spatial == dim
        self.d_temporal = d_temporal
        self.d_spatial = d_spatial
        # Standard RoPE frequencies for all dim//2 slots
        inv_freq = 1.0 / (base ** (torch.arange(0, dim, 2).float() / dim))
        self.register_buffer("inv_freq", inv_freq)

    def forward(self, position_ids):
        """position_ids: (batch, seq_len, 3) -> (m, x, y)"""
        m = position_ids[..., 0]  # (batch, seq_len)
        x = position_ids[..., 1]
        y = position_ids[..., 2]

        # Temporal angles: first d_temporal//2 frequency slots
        freq_t = self.inv_freq[:self.d_temporal // 2]
        angles_t = m.unsqueeze(-1) * freq_t  # (batch, seq, d_t//2)

        # Spatial angles: last d_spatial//2 frequency slots, interleaved
        freq_s = self.inv_freq[self.d_temporal // 2:]
        angles_x = x.unsqueeze(-1) * freq_s[0::2]  # even spatial slots
        angles_y = y.unsqueeze(-1) * freq_s[1::2]  # odd spatial slots
        # Interleave x and y
        angles_s = torch.stack([angles_x, angles_y], dim=-1).flatten(-2)

        angles = torch.cat([angles_t, angles_s], dim=-1)  # (batch, seq, dim//2)
        cos = angles.cos()
        sin = angles.sin()
        return cos, sin
```

4. Build the Chebyshev causal mask:

```python
def build_chebyshev_mask(position_ids, image_token_mask):
    """Chebyshev causal mask for visual self-attention."""
    x = position_ids[..., 1]  # (batch, seq_len)
    y = position_ids[..., 2]
    d_cheb = torch.max(x.abs(), y.abs())  # (batch, seq_len)

    # For visual tokens: allow attention to tokens with <= Chebyshev distance
    # d_cheb[i] >= d_cheb[j] means token i can attend to token j
    vis_mask = d_cheb.unsqueeze(-1) >= d_cheb.unsqueeze(-2)  # (batch, seq, seq)

    # Standard causal mask for non-visual tokens
    seq_len = position_ids.shape[1]
    causal = torch.tril(torch.ones(seq_len, seq_len, dtype=torch.bool))

    # Combine: use Chebyshev where both tokens are visual, causal elsewhere
    both_visual = image_token_mask.unsqueeze(-1) & image_token_mask.unsqueeze(-2)
    mask = torch.where(both_visual, vis_mask, causal.unsqueeze(0))
    return mask
```

Output: The model now assigns spatially-aware positional encodings to image patches and uses center-outward causal masking for visual attention, improving spatial reasoning on 3D scene QA benchmarks.

---

**Example 2: Diagnosing and fixing attention decay on early image tokens**

User: "My vision-language model ignores details from the first image when given 16 multi-view images. Attention visualization shows the first few images get almost no attention from later tokens."

Approach:
1. Confirm the root cause: standard RoPE's temporal decay means tokens 5000+ positions away receive exponentially less attention, and the first image's tokens start at position ~0 while later text is at position ~10000+.
2. Implement C²RoPE's spatial encoding so that within each image, spatial relationships dominate over temporal distance.
3. Apply Chebyshev masking per-image so each image's internal attention structure is spatially rather than temporally ordered.
4. Between images, the temporal index still provides ordering, but the spatial dimensions being zero for text tokens means cross-modal attention falls back to standard RoPE behavior without the destructive column-discontinuity artifacts.

Output: After applying C²RoPE, attention maps show that early images receive comparable attention weight to later images for visual self-attention, and spatial questions ("what is next to the sofa in view 1?") improve significantly.

---

**Example 3: Minimal patch -- adding spatial continuity without Chebyshev masking**

User: "I want the spatial continuity fix but don't want to change the attention mask. Can I use just the triplet positional encoding part?"

Approach:
1. Yes -- the triplet (m, x, y) positional encoding and Chebyshev masking are independent components.
2. Implement only steps 1-5 from the workflow (frequency allocation and triplet index).
3. Keep the standard causal mask unchanged.
4. This alone fixes the column discontinuity problem where vertically adjacent patches had distant position indices. It does not fix the long-range attention decay issue, which requires the Chebyshev mask.

```python
# Minimal change: just replace position_ids construction
# In your model's prepare_inputs method:
position_ids = build_c2rope_position_ids(input_ids, IMAGE_TOKEN_ID)
cos, sin = c2rope_embed(position_ids)
# Apply cos, sin to Q, K as usual -- no mask changes needed
```

Output: Spatial reasoning improves partially (column-adjacency artifacts eliminated), but long-range visual token neglect persists without Chebyshev masking.

## Best Practices

- **Do:** Keep the temporal dimension allocation larger than spatial (e.g., 96 vs 32 for d=128). The pretrained LLM relies heavily on temporal position; allocating too many dimensions to spatial coordinates degrades text understanding.
- **Do:** Center spatial coordinates at (0, 0) for the image center. This ensures the Chebyshev mask creates symmetric rings and the coordinate magnitudes stay small relative to temporal indices.
- **Do:** Set x=0, y=0 for all text tokens so they behave identically to standard RoPE. This preserves pretrained text capabilities without any fine-tuning.
- **Do:** Apply Chebyshev masking only within individual images. Cross-image and text-image attention should use standard causal masking.
- **Avoid:** Assigning spatial coordinates that exceed the temporal index range. If your image grid is 24x24, coordinates range roughly from -12 to +12, while temporal indices may reach 10000+. The frequency allocation strategy handles this naturally, but manually scaling coordinates to match temporal magnitude will break the separation.
- **Avoid:** Using this for 1D sequential data (audio waveforms, pure text). C²RoPE's benefit comes entirely from exploiting 2D spatial structure in visual tokens. For non-visual modalities, standard RoPE is correct.

## Error Handling

- **Dimension mismatch:** If d_temporal + d_spatial != d, the frequency allocation will silently produce wrong results. Assert this equality at initialization.
- **Non-square image grids:** The coordinate formula works for rectangular grids (H != W). Ensure you use the correct H and W when computing (x, y) rather than assuming sqrt(n_patches).
- **Mixed-precision training:** The interleaved sin/cos computation for spatial dimensions can accumulate floating point error in fp16. Compute rotation angles in fp32 and cast back, consistent with standard RoPE best practices.
- **Variable image sizes:** If images in a batch have different patch counts, you need per-image coordinate computation. Pad the position_ids tensor and mask appropriately.
- **Pretrained weight compatibility:** C²RoPE does not add new learnable parameters -- it only changes how position indices are constructed. Pretrained weights load without modification, but a short fine-tuning phase (the paper uses standard LLaVA training) helps the model adapt to the new positional structure.

## Limitations

- **2D visual data only.** The triplet index assumes a 2D grid of patches. For video with a true temporal axis, you would need a 4-tuple (m, x, y, t_frame) -- the paper does not address this.
- **Center-outward causality assumption.** Chebyshev masking assumes the image center is the most "fundamental" region. This is a reasonable default but may not hold for all visual tasks (e.g., peripheral detection tasks, document OCR where reading order matters).
- **No benefit for text-only tasks.** Since text tokens use x=0, y=0, the encoding collapses to standard RoPE for text. There is zero benefit and minor computational overhead for text-only workloads.
- **Fixed frequency split.** The 96/32 allocation is a hyperparameter tuned for d=128. For other dimensions, the optimal split is not established and would require experimentation.
- **Training required for full gains.** While the encoding is compatible with pretrained weights, the full benchmark improvements (e.g., +18.1 CIDEr on ScanQA) require fine-tuning on downstream tasks.

## Reference

**Paper:** [C²RoPE: Causal Continuous Rotary Positional Encoding for 3D Large Multimodal-Models Reasoning](https://arxiv.org/abs/2602.10551v1) (ICRA 2026)
**Code:** [github.com/ErikZ719/C2RoPE](https://github.com/ErikZ719/C2RoPE)
**Key sections to study:** Section 3 (Method) for the triplet index construction and frequency allocation formulas; Section 3.3 for Chebyshev Causal Masking derivation; Table 1 for benchmark comparisons showing +4.3 EM@1 on ScanQA and +1.2 on SQA3D over the LLaVA-3D baseline.