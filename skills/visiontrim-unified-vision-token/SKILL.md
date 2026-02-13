---
name: "visiontrim-unified-vision-token"
description: "Implement VisionTrim's training-free visual token compression for multimodal LLMs. Combines attention-based dominant token selection (DVTS) with text-guided token merging (TGVC) to reduce vision token counts by 75-90% with minimal accuracy loss. Use when: 'compress vision tokens in my MLLM', 'speed up LLaVA inference', 'reduce visual token overhead', 'implement token pruning for multimodal model', 'optimize MLLM for deployment', 'add training-free vision compression'."
---

# VisionTrim: Training-Free Vision Token Compression for MLLMs

This skill enables Claude to implement VisionTrim, a plug-and-play framework that compresses visual tokens in multimodal large language models (MLLMs) without retraining. VisionTrim pairs two modules: Dominant Vision Token Selection (DVTS), which uses CLS-token attention and local affinity to keep the most informative image patches, and Text-Guided Vision Complement (TGVC), which recovers task-relevant tokens that DVTS missed by scoring them against the text query via CLIP similarity and merging them through iterative clustering. Together they cut vision token counts from 576+ down to ~64 while preserving benchmark accuracy within 1-2 points.

## When to Use

- When a user wants to reduce inference latency of a LLaVA or similar vision-language model by compressing visual tokens
- When deploying an MLLM to resource-constrained environments (edge devices, limited GPU memory) and need to shrink the vision token sequence
- When building a video understanding pipeline where per-frame token counts make multi-frame processing prohibitively expensive
- When the user asks to implement attention-based token pruning or token merging in a vision transformer encoder
- When integrating a training-free acceleration method into an existing MLLM codebase without modifying model weights
- When comparing token compression strategies (FastV, ToMe, SparseVLM) and wanting VisionTrim as a stronger baseline

## Key Technique

**DVTS (Dominant Vision Token Selection)** operates inside the vision encoder (e.g., CLIP ViT). It extracts attention maps from the penultimate transformer layer, specifically the CLS token's attention to all image patch tokens. It then builds a local affinity graph between neighboring patches to capture spatial structure. Three signals are combined with adaptive weighting: (1) CLS attention, measuring global saliency; (2) local affinity importance, capturing fine-grained spatial relevance; (3) average received attention from all tokens. The adaptive weighting uses inverse variance: if CLS attention is confident (low variance), it gets higher weight; if local affinity is more discriminative, it dominates. The top-k tokens by this combined score are selected (default: 48).

**TGVC (Text-Guided Vision Complement)** addresses DVTS's blind spot: tokens that are visually unremarkable but semantically critical for the user's question. It encodes the text query through CLIP's text encoder and computes cosine similarity between the text embedding and every non-selected image token. The highest-scoring tokens (default: 16) become cluster centers. An iterative assignment-aggregation loop (3 rounds) then merges remaining discarded tokens into these centers based on hidden-state similarity, producing compact merged representations. This ensures the final token set is aligned with what the text query actually needs.

The two modules are complementary: DVTS provides a strong vision-centric foundation, TGVC adds text-aware recovery. Both are plug-and-play -- they modify only the vision encoder's forward pass and require zero weight changes.

## Step-by-Step Workflow

1. **Install the VisionTrim environment.** Clone `https://github.com/hanxunyu/VisionTrim`, create a Python 3.10 conda environment, run `pip install -e .` and `pip install protobuf`. Optionally install `flash-attn` for additional speedup.

2. **Identify the target MLLM architecture.** VisionTrim's reference implementation supports LLaVA-1.5 (7B/13B) and LLaVA-1.6-NeXT (7B/13B). For other architectures (Qwen-VL, InternVL), you will adapt the same two-module pattern to their vision encoder.

3. **Inject DVTS into the vision encoder's forward pass.** In the CLIP encoder (or equivalent ViT), capture multi-head attention outputs from the penultimate layer. Compute CLS-to-patch attention by averaging across heads: `cls_attn = attn_weights[:, :, 0, 1:].mean(dim=1)`. Build a local affinity matrix with `build_token_affinity()` using spatial neighbor relationships on the patch grid.

4. **Combine importance scores with adaptive weighting.** Compute variance of CLS attention and local affinity scores. Weight them inversely: `cls_weight = local_var / (cls_var + local_var)`. Add average received attention as a third signal. Merge into a single importance score per token.

5. **Select dominant tokens via top-k.** Use `torch.topk(combined_score, DVTS_token_num)` to retain the most important tokens. Default `DVTS_token_num=48` compresses 576 tokens down to 48 (92% reduction). Adjust based on your accuracy-speed tradeoff.

6. **Implement TGVC for text-guided recovery.** Encode the text prompt through CLIP's text encoder. Compute scaled cosine similarity between the text embedding and every non-selected image token: `sim = matmul(text_norm, img_norm.T) * logit_scale.exp()`. Average across text token positions to get one score per image token.

7. **Select complement tokens and merge via iterative clustering.** Take the top-k non-selected tokens by text similarity (`TGVC_token_num=16` default) as cluster centers. For 3 iterations: assign remaining discarded tokens to nearest center by hidden-state similarity, aggregate assigned tokens into each center via weighted averaging, update center representations.

8. **Concatenate and pass to the LLM.** The final vision representation is `[DVTS_selected_tokens, TGVC_merged_tokens]`, yielding 48+16=64 tokens (from 576 original). Prepend/interleave with text tokens per the model's standard input format.

9. **Tune hyperparameters for your use case.** Run evaluation on your target benchmark varying `DVTS_token_num` in {32, 48, 64, 96} and `TGVC_token_num` in {8, 16, 24}. Higher values preserve more accuracy; lower values give more speedup. For video tasks with many frames, use aggressive settings (32+8).

10. **Evaluate and compare.** Run the provided benchmark scripts: `bash scripts/v1_5/eval/{benchmark}.sh $DVTS_token_num $TGVC_token_num`. Compare against the uncompressed baseline and other methods (FastV, ToMe) to validate your configuration.

## Concrete Examples

**Example 1: Adding VisionTrim to a LLaVA-1.5 deployment**

User: "I'm running LLaVA-1.5-7B for image QA but inference is too slow. Help me add VisionTrim to compress the visual tokens."

Approach:
1. Clone VisionTrim repo and install into the existing environment
2. Replace the standard CLIP encoder with VisionTrim's modified `clip_encoder.py`
3. Set method to `VisionTrim` in model config and pass `DVTS_token_num=48`, `TGVC_token_num=16`
4. The text query is automatically forwarded to TGVC via `original_qs` parameter
5. Run evaluation on target benchmarks to validate quality

```python
# In model initialization
model.config.method = "VisionTrim"
model.config.DVTS_token_num = 48   # Keep 48 dominant tokens from 576
model.config.TGVC_token_num = 16   # Merge 16 text-relevant complement tokens

# Forward pass automatically applies:
# 576 vision tokens -> 48 (DVTS) + 16 (TGVC) = 64 tokens (89% reduction)
```

Output: Inference processes 64 vision tokens instead of 576, yielding ~4-5x speedup in the LLM decoder with <1.5% accuracy drop on GQA, TextVQA, and POPE.

**Example 2: Implementing DVTS token selection from scratch for a custom ViT**

User: "I have a custom vision-language model using a ViT-L encoder. Show me how to implement the DVTS attention-based token selection."

Approach:
1. Hook into the penultimate ViT layer to extract attention weights
2. Compute the three importance signals
3. Apply adaptive weighting and top-k selection

```python
import torch

def dvts_select(attn_output, num_tokens=48):
    """
    attn_output: attention weights from penultimate ViT layer
                 shape: (batch, num_heads, seq_len, seq_len)
    Returns: indices of selected tokens, shape (batch, num_tokens)
    """
    # 1. CLS-to-patch attention (global saliency)
    cls_attn = attn_output[:, :, 0, 1:].mean(dim=1)  # (B, num_patches)

    # 2. Average received attention (how much other tokens attend to each patch)
    patch_attn = attn_output[:, :, 1:, 1:]  # exclude CLS row and col
    avg_received = patch_attn.mean(dim=(1, 2))  # (B, num_patches)

    # 3. Local affinity importance (spatial neighbor analysis)
    H = W = int(cls_attn.shape[1] ** 0.5)  # assume square grid
    local_importance = compute_local_affinity(cls_attn.view(-1, H, W))

    # 4. Adaptive weighting by inverse variance
    cls_var = cls_attn.var(dim=1, keepdim=True).clamp(min=1e-8)
    local_var = local_importance.var(dim=1, keepdim=True).clamp(min=1e-8)
    cls_w = local_var / (cls_var + local_var)
    local_w = cls_var / (cls_var + local_var)

    combined = cls_w * cls_attn + local_w * local_importance + 0.1 * avg_received
    _, indices = torch.topk(combined, num_tokens, dim=1)
    return indices

def compute_local_affinity(attn_map_2d):
    """Compute importance based on local 3x3 neighborhood variance."""
    # Unfold into 3x3 patches and compute local variance
    B, H, W = attn_map_2d.shape
    padded = torch.nn.functional.pad(attn_map_2d, (1,1,1,1), mode='reflect')
    unfolded = padded.unfold(1, 3, 1).unfold(2, 3, 1)  # (B, H, W, 3, 3)
    local_var = unfolded.reshape(B, H, W, 9).var(dim=-1)
    return local_var.reshape(B, H * W)
```

**Example 3: Adding TGVC text-guided merging to an existing token pruning pipeline**

User: "I already prune vision tokens with attention scores but lose accuracy on text-heavy images. Help me add text-guided recovery like TGVC."

Approach:
1. Encode the text query through CLIP's text encoder
2. Score discarded tokens against the text embedding
3. Use iterative clustering to merge discarded tokens into compact representations

```python
import torch
import torch.nn.functional as F

def tgvc_complement(text_features, selected_indices, all_hidden_states,
                    clip_logit_scale, num_complement=16, num_iters=3):
    """
    text_features: CLIP text encoding, (B, text_len, D)
    selected_indices: indices already kept by DVTS, (B, K)
    all_hidden_states: full vision hidden states, (B, N, D)
    Returns: merged complement tokens, (B, num_complement, D)
    """
    B, N, D = all_hidden_states.shape

    # Mask out already-selected tokens
    mask = torch.ones(B, N, dtype=torch.bool, device=all_hidden_states.device)
    mask.scatter_(1, selected_indices, False)
    other_hidden = all_hidden_states[mask].view(B, -1, D)  # (B, N-K, D)

    # Score discarded tokens against text query
    text_norm = F.normalize(text_features, dim=-1)
    other_norm = F.normalize(other_hidden, dim=-1)
    sim = torch.matmul(text_norm, other_norm.transpose(-2, -1)) * clip_logit_scale.exp()
    token_scores = sim.mean(dim=1)  # (B, N-K)

    # Select top complement tokens as cluster centers
    _, comp_idx = torch.topk(token_scores, num_complement, dim=1)
    centers = torch.gather(other_hidden, 1, comp_idx.unsqueeze(-1).expand(-1,-1,D))

    # Iterative merge: assign remaining to centers, aggregate
    remaining_mask = torch.ones_like(token_scores, dtype=torch.bool)
    remaining_mask.scatter_(1, comp_idx, False)
    to_merge = other_hidden[remaining_mask].view(B, -1, D)

    for _ in range(num_iters):
        # Cosine similarity assignment
        center_norm = F.normalize(centers, dim=-1)
        merge_norm = F.normalize(to_merge, dim=-1)
        assign_sim = torch.matmul(merge_norm, center_norm.transpose(-2, -1))
        assign_idx = assign_sim.argmax(dim=-1)  # (B, M)

        # Aggregate into centers
        one_hot = F.one_hot(assign_idx, num_complement).float()  # (B, M, C)
        counts = one_hot.sum(dim=1, keepdim=True).transpose(1, 2).clamp(min=1)
        aggregated = torch.matmul(one_hot.transpose(1, 2), to_merge) / counts
        centers = centers + 0.5 * (aggregated - centers)  # smooth update

    return centers
```

Output: The 16 merged complement tokens capture text-relevant visual details (e.g., text on signs, specific objects mentioned in the query) that pure attention pruning would discard.

## Best Practices

- **Do:** Start with the default configuration (`DVTS_token_num=48`, `TGVC_token_num=16`) and adjust only after measuring accuracy on your target task. These defaults preserve >98% accuracy on most benchmarks.
- **Do:** Use the penultimate layer's attention (not the last layer) for DVTS -- the last layer's attention tends to be overly diffuse and less discriminative.
- **Do:** Forward the actual user text query to TGVC, not a generic placeholder. The text guidance is only useful when it reflects the real task.
- **Do:** For video MLLMs, apply VisionTrim per-frame with aggressive settings (32+8 = 40 tokens/frame) to keep total token count manageable across many frames.
- **Avoid:** Setting `DVTS_token_num` below 32 for complex scene understanding tasks -- this drops too many spatial details for benchmarks like TextVQA that require reading fine text.
- **Avoid:** Skipping TGVC and using only DVTS. The text-guided complement recovers 1-3 accuracy points on text-heavy benchmarks (TextVQA, OCR tasks) that pure attention pruning misses.

## Error Handling

- **CLIP text encoder unavailable:** If your MLLM doesn't use CLIP as the vision encoder, TGVC's text similarity scoring needs adaptation. Use whatever shared embedding space exists between your text and vision encoders, or fall back to DVTS-only mode (still effective, just less text-aware).
- **Non-square patch grids:** The local affinity computation in DVTS assumes a spatial grid. For non-square inputs (e.g., high-resolution LLaVA-NeXT with dynamic aspect ratios), compute H and W from the actual image dimensions rather than assuming `sqrt(N)`.
- **Attention output shape mismatch:** Some ViT implementations don't return attention weights by default. Ensure `output_attentions=True` is set in the vision encoder config. For models using FlashAttention (which doesn't natively return attention maps), you may need to run one pass without FlashAttention for the penultimate layer or use an approximation.
- **Token index alignment after selection:** When concatenating DVTS and TGVC tokens, ensure positional encodings match the original token positions. If the LLM uses absolute position embeddings, gather the corresponding position embeddings for selected indices rather than using sequential positions.

## Limitations

- **Architecture coupling:** The reference implementation is tightly integrated with LLaVA's codebase. Porting to architectures with different vision-language fusion strategies (cross-attention models like Flamingo, early-fusion models) requires rethinking where token compression is applied.
- **Text dependency for TGVC:** TGVC requires the text query at vision encoding time. This doesn't work for systems where vision features are precomputed and cached independently of queries (e.g., retrieval-augmented setups).
- **Static compression ratio:** The same number of tokens is kept regardless of image complexity. A blank wall and a dense infographic both produce 64 tokens. Adaptive token budgeting (varying K per image) is not implemented.
- **No training-time benefit:** VisionTrim accelerates inference only. Training still processes full token sequences. It is not applicable to reducing training compute.
- **Benchmark-specific tuning:** Optimal DVTS/TGVC ratios vary across tasks. Settings that work well for visual QA (GQA) may underperform on OCR-heavy tasks (TextVQA) and vice versa.

## Reference

**Paper:** [VisionTrim: Unified Vision Token Compression for Training-Free MLLM Acceleration](https://arxiv.org/abs/2601.22674v2) (ICLR 2026). Look for Section 3 (Method) for the full DVTS importance scoring formula and TGVC iterative merging algorithm, and Tables 1-3 for benchmark comparisons showing VisionTrim outperforming FastV, ToMe, and SparseVLM across 8 benchmarks.

**Code:** [https://github.com/hanxunyu/VisionTrim](https://github.com/hanxunyu/VisionTrim) -- `clip_encoder.py` contains the core DVTS and TGVC implementations.