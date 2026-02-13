---
name: "vica-multimodal-vision-only-cross-attention"
description: "Implement and apply the ViCA (Vision-only Cross-Attention) architecture to reduce visual computation in multimodal LLMs to ~4% while retaining 98% accuracy. Use when: 'optimize multimodal model inference', 'reduce vision token overhead in MLLM', 'implement sparse cross-attention for vision-language', 'speed up multimodal inference', 'apply ViCA architecture to my model', 'convert dense visual attention to sparse cross-attention'."
---

# ViCA: Vision-only Cross-Attention for Efficient Multimodal LLMs

This skill enables Claude to help users implement, adapt, and debug the ViCA architecture pattern — a technique that removes visual tokens from all self-attention and feed-forward layers in multimodal LLMs, restricting vision-language interaction to sparse cross-attention at a small subset of diagnostically selected layers. ViCA reduces visual-side computation to ~4% of the baseline while preserving 98% accuracy, yielding 3.5x single-batch and 10x+ multi-batch inference speedups.

## When to Use

- When a user wants to reduce inference latency of a multimodal LLM (LLaVA, Qwen-VL, or similar) without significant accuracy loss
- When implementing a new MLLM architecture and deciding how visual tokens should interact with text across Transformer layers
- When a user asks how to identify which layers in their model actually perform meaningful vision-language fusion
- When applying token efficiency techniques and the user wants to combine ViCA with token pruning (e.g., PyramidDrop, FastV)
- When porting an existing unified self-attention MLLM to a cross-attention design for production deployment
- When profiling a multimodal model and discovering that visual token processing dominates latency

## Key Technique

### The Core Insight: Visual Embeddings Are Already Aligned

Standard MLLMs (LLaVA, Qwen-VL, InternVL) process visual and text tokens jointly through every Transformer layer via unified self-attention. ViCA challenges this by demonstrating that **projected visual embeddings are already well-aligned with the language embedding space** after the vision encoder + projection layer. Freezing all visual token updates (self-attention and FFN) across all layers causes only marginal accuracy drops — meaning the repeated refinement of visual representations through dozens of Transformer layers provides negligible benefit.

### Sparse Cross-Attention at Selected Layers

Rather than processing visual tokens through every layer, ViCA restructures the architecture so that visual tokens **bypass all self-attention (V2V, V2T) and feed-forward computations entirely**. Text tokens attend to visual tokens only through asymmetric cross-attention at a small number of diagnostically selected layers. In this cross-attention, text tokens provide the queries, while keys and values come from both text and the frozen visual embeddings. The selection of which layers receive cross-attention is determined by a two-metric diagnostic: (1) **cosine similarity** between token embeddings before and after each module (measuring representation change magnitude), and (2) **KL divergence** on the output distribution (measuring actual prediction impact). For LLaVA-1.5-7B, this identifies layers 0-1, 7-11, and 14 as critical — roughly 8 out of 32 layers.

### Hardware-Friendly and Composable

Because visual tokens no longer participate in self-attention or FFN, the KV projections for visual tokens can be precomputed once and cached. The resulting inference pipeline is regular and compatible with FlashAttention v2.1+ (using bottom-right causal masking). ViCA is orthogonal to token pruning methods — combining it with PyramidDrop achieves 2% visual computation at 96.3% accuracy.

## Step-by-Step Workflow

1. **Identify the base MLLM architecture.** Determine the model family (LLaVA-1.5, Qwen-VL, InternVL, or custom), the number of Transformer layers, the vision encoder (CLIP ViT, SigLIP, etc.), and the projection module (MLP, Q-Former, etc.) that maps visual features into the language embedding space.

2. **Run layer importance diagnostics.** For each Transformer layer `l`, compute two metrics over a small calibration set (~500 samples from the training distribution):
   - Cosine similarity: `cos_sim(h_before_module, h_after_module)` for visual tokens — high similarity means the layer barely modifies visual representations.
   - KL divergence: `KL(p_full || p_masked)` where `p_masked` is the output distribution when vision-text interaction is removed at layer `l` — high KL means the layer is critical for multimodal reasoning.

3. **Select cross-attention layers.** Rank layers by KL divergence (descending). Choose the top-K layers where removing vision-text interaction causes the largest output distribution shift. Typical values: K=6-10 for a 32-layer model. Validate by checking that the selected set covers early layers (alignment), middle layers (reasoning), and at least one late layer (grounding).

4. **Modify the model architecture.** For each Transformer layer:
   - **Non-selected layers:** Remove visual tokens from the attention computation entirely. Text tokens attend only to other text tokens via standard causal self-attention. Visual tokens receive no updates.
   - **Selected layers:** Replace unified self-attention with asymmetric cross-attention where `Q = text_tokens @ W_q`, `K = concat(text_tokens, visual_tokens) @ W_k`, `V = concat(text_tokens, visual_tokens) @ W_v`. Visual tokens still receive no write-back updates (no `delta_V`).
   - **All layers:** Remove visual tokens from the FFN pathway completely.

5. **Precompute and cache visual KV projections.** Since visual tokens are frozen after the projection layer, compute `K_vis = visual_embeddings @ W_k` and `V_vis = visual_embeddings @ W_v` once for each selected cross-attention layer at the start of inference. Store these as static KV cache entries.

6. **Adapt attention masking for FlashAttention.** Configure the causal mask with `causal=True` and set the mask alignment to bottom-right corner to properly handle the asymmetric attention pattern where text queries attend to all visual keys (non-causal) and prior text keys (causal).

7. **Fine-tune the modified model.** Use the same two-stage training protocol as the base model (pretraining on captioning data, then instruction tuning), but with the ViCA architecture active from the start. Typical fine-tuning on LLaVA-665K instruction data recovers any accuracy gap from the architectural change.

8. **Benchmark against baselines.** Evaluate on standard multimodal benchmarks (MMBench, GQA, VQAv2, TextVQA, POPE, MME-P) and compare against the base MLLM, FastV, PyramidDrop, and other pruning methods. Measure both accuracy retention and actual wall-clock speedup.

9. **Optionally combine with token pruning.** Apply a token pruning method (PyramidDrop, FastV) on top of ViCA to further reduce the number of visual tokens at selected cross-attention layers, achieving compound efficiency gains (e.g., 4% computation from ViCA x 50% token reduction = 2% total visual computation).

10. **Profile and deploy.** Measure end-to-end latency for single-batch and multi-batch scenarios. Verify that visual grounding overhead is near-zero compared to text-only inference. Use the regular compute pattern for efficient batching and serving.

## Concrete Examples

**Example 1: Converting LLaVA-1.5-7B to ViCA**

User: "I have a LLaVA-1.5-7B model and inference is too slow for production. How do I apply ViCA?"

Approach:
1. Start with the 32-layer Vicuna-7B backbone and CLIP-ViT-L/14 vision encoder
2. Run the diagnostic on 500 samples from LLaVA-665K to identify critical layers
3. Select layers {0, 1, 7, 8, 9, 10, 11, 14} for cross-attention (8 of 32 layers)
4. Modify the model forward pass:

```python
class ViCATransformerLayer(nn.Module):
    def __init__(self, base_layer, layer_idx, cross_attn_layers):
        super().__init__()
        self.base_layer = base_layer
        self.is_cross_attn = layer_idx in cross_attn_layers

    def forward(self, text_hidden, visual_kv_cache=None):
        # Text-only self-attention (visual tokens never enter here)
        text_out = self.base_layer.self_attn(
            query=text_hidden, key=text_hidden, value=text_hidden,
            causal=True
        )

        if self.is_cross_attn and visual_kv_cache is not None:
            # Asymmetric cross-attention: text queries, visual+text keys/values
            k_vis, v_vis = visual_kv_cache[self.layer_idx]
            k_combined = torch.cat([k_vis, self.base_layer.k_proj(text_hidden)], dim=1)
            v_combined = torch.cat([v_vis, self.base_layer.v_proj(text_hidden)], dim=1)
            text_out = text_out + self.cross_attn(
                query=text_hidden, key=k_combined, value=v_combined
            )

        # FFN on text tokens only (visual tokens are never processed)
        text_out = self.base_layer.ffn(text_out)
        return text_out
```

5. Precompute visual KV cache once per image:

```python
def precompute_visual_kv(visual_embeddings, model, cross_attn_layers):
    """Compute K, V projections for visual tokens at selected layers."""
    kv_cache = {}
    for layer_idx in cross_attn_layers:
        layer = model.layers[layer_idx]
        kv_cache[layer_idx] = (
            layer.k_proj(visual_embeddings),  # [B, N_vis, D]
            layer.v_proj(visual_embeddings),  # [B, N_vis, D]
        )
    return kv_cache
```

Result: 3.5x faster single-batch inference, 97.8% of original accuracy across 9 benchmarks.

**Example 2: Layer Importance Diagnostic Script**

User: "How do I find which layers in my multimodal model actually need vision-text interaction?"

Approach:
1. Run calibration data through the full model, recording hidden states
2. Compute per-layer importance metrics
3. Rank and select layers

```python
import torch
from torch.nn.functional import cosine_similarity, kl_div, log_softmax, softmax

def diagnose_layer_importance(model, dataloader, num_layers):
    """Identify layers where vision-language interaction matters most."""
    cos_scores = torch.zeros(num_layers)
    kl_scores = torch.zeros(num_layers)

    # Get baseline output distribution with full model
    with torch.no_grad():
        for batch in dataloader:
            logits_full = model(**batch).logits
            p_full = softmax(logits_full[:, -1, :], dim=-1)

            for layer_idx in range(num_layers):
                # Metric 1: Cosine similarity of visual tokens before/after layer
                h_before = model.get_hidden_before_layer(batch, layer_idx)
                h_after = model.get_hidden_after_layer(batch, layer_idx)
                vis_mask = batch["visual_token_mask"]
                cos = cosine_similarity(
                    h_before[vis_mask], h_after[vis_mask], dim=-1
                ).mean()
                cos_scores[layer_idx] += cos

                # Metric 2: KL divergence when masking vision at this layer
                logits_masked = model.forward_mask_vision_at_layer(
                    batch, layer_idx
                ).logits
                p_masked = softmax(logits_masked[:, -1, :], dim=-1)
                kl = kl_div(
                    log_softmax(logits_masked[:, -1, :], dim=-1),
                    p_full, reduction="batchmean"
                )
                kl_scores[layer_idx] += kl

    # Normalize
    cos_scores /= len(dataloader)
    kl_scores /= len(dataloader)

    # Select layers with highest KL divergence (most impact when removed)
    selected = torch.argsort(kl_scores, descending=True)[:8].sort().values
    return selected.tolist(), kl_scores, cos_scores
```

Output: `Selected cross-attention layers: [0, 1, 7, 8, 9, 10, 11, 14]`

**Example 3: Combining ViCA with Token Pruning**

User: "I already use PyramidDrop for token pruning. Can I stack ViCA on top?"

Approach:
1. Apply ViCA architecture (visual tokens bypass self-attn/FFN, sparse cross-attention at selected layers)
2. At each cross-attention layer, apply PyramidDrop to progressively reduce the number of visual tokens
3. The two methods are orthogonal — ViCA reduces which layers process vision, PyramidDrop reduces how many tokens per layer

```python
class ViCAWithPyramidDrop(nn.Module):
    def forward(self, text_hidden, visual_embeddings):
        visual_kv = self.precompute_visual_kv(visual_embeddings)

        for layer_idx, layer in enumerate(self.layers):
            if layer_idx in self.cross_attn_layers:
                # Apply PyramidDrop: prune visual tokens at this stage
                k_vis, v_vis = visual_kv[layer_idx]
                keep_ratio = self.pyramid_schedule[layer_idx]  # e.g., 1.0 -> 0.5 -> 0.25
                k_vis, v_vis = self.prune_tokens(k_vis, v_vis, keep_ratio)

                text_hidden = layer(text_hidden, visual_kv=(k_vis, v_vis))
            else:
                text_hidden = layer(text_hidden, visual_kv=None)

        return text_hidden
```

Result: ~2% total visual computation, 96.3% baseline accuracy — near text-only inference speed with multimodal capability.

## Best Practices

- **Do:** Run the full layer diagnostic before selecting cross-attention layers. The optimal set varies by model architecture and size — don't assume LLaVA-7B layer indices transfer to a 13B or 72B model.
- **Do:** Precompute visual KV projections once per image and cache them. This is the key to achieving near-zero visual overhead during autoregressive text decoding.
- **Do:** Fine-tune with the ViCA architecture active from the start of training, not applied post-hoc. Training-aware ViCA recovers ~2% more accuracy than training-free application.
- **Do:** Use FlashAttention v2.1+ with bottom-right causal mask alignment for the asymmetric attention pattern. This avoids materializing the full attention matrix.
- **Avoid:** Applying ViCA without the diagnostic step. Uniformly spacing cross-attention layers performs significantly worse than importance-based selection.
- **Avoid:** Processing visual tokens through FFN layers "just in case." The paper shows FFN processing of visual tokens contributes near-zero benefit after projection alignment, and removing it is essential for the speedup.
- **Avoid:** Using fewer than 6 cross-attention layers for 7B-scale models. Below this threshold, accuracy drops become nonlinear and harder to recover through fine-tuning.

## Error Handling

- **Accuracy drops beyond 5%:** Re-run the layer diagnostic with a larger calibration set (1000+ samples). Check that the vision encoder projection is properly trained — ViCA assumes good initial alignment. If alignment is poor, add 1-2 more cross-attention layers in early positions.
- **No speedup observed:** Verify that visual tokens are truly excluded from self-attention and FFN computation, not just masked. Masking still allocates memory and compute. The forward pass must structurally skip visual tokens at non-selected layers.
- **FlashAttention compatibility errors:** Ensure the causal mask is set to bottom-right alignment. The asymmetric attention (text queries attending to visual+text keys) requires this configuration. Fall back to standard attention with an explicit mask if FlashAttention version is < 2.1.
- **OOM during diagnostic:** The layer diagnostic requires running the model with per-layer hooks. Use gradient checkpointing or reduce calibration batch size. The diagnostic itself doesn't require gradients — always wrap in `torch.no_grad()`.
- **Multi-image inputs:** ViCA's visual KV cache scales linearly with the number of images. For multi-image scenarios, concatenate visual embeddings from all images before KV projection. Monitor memory if image count is large.

## Limitations

- **Requires fine-tuning for best results.** Training-free ViCA (applied as inference-time modification) retains ~92-95% accuracy. The full 98% recovery requires fine-tuning on instruction data, which needs GPU resources and the original training data.
- **Layer selection is model-specific.** The optimal cross-attention layer set must be re-diagnosed for each model family, size, and even checkpoint. There is no universal "one-size-fits-all" layer configuration.
- **Not suitable for vision-heavy tasks requiring fine-grained spatial reasoning.** Tasks like dense object detection, pixel-level segmentation, or detailed OCR may need more visual refinement than ViCA's frozen embeddings provide.
- **Assumes a well-trained vision projection.** If the MLP/Q-Former projecting visual features into language space is undertrained, the core assumption (visual embeddings are already aligned) breaks down, and ViCA will underperform.
- **Repository is minimal.** As of early 2026, the official EIT-NLP/ViCA repository contains only a license and README. Implementation must currently be done from the paper's descriptions.

## Reference

- **Paper:** [ViCA: Efficient Multimodal LLMs with Vision-Only Cross-Attention](https://arxiv.org/abs/2602.07574v1) (Liu et al., 2026). Focus on Section 3 (method) for the architectural modification, Section 3.2 for the layer diagnostic procedure, and Section 4.2 for the ablation study showing how layer count and selection strategy affect accuracy-efficiency tradeoffs.
- **Repository:** [https://github.com/EIT-NLP/ViCA](https://github.com/EIT-NLP/ViCA) (Apache-2.0, minimal as of Feb 2026)