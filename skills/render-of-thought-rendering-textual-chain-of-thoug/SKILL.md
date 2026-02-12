---
name: "render-of-thought-rendering-textual-chain-of-thoug"
description: |
  Implement the Render-of-Thought (RoT) framework for compressing verbose Chain-of-Thought reasoning into dense visual-latent embeddings.
  Uses rendered text images as training-time supervision to structure a latent reasoning space, then eliminates rendering at inference for 3-4x token compression.
  Trigger phrases:
  - "compress chain-of-thought reasoning"
  - "render-of-thought" or "RoT reasoning"
  - "reduce CoT token overhead"
  - "visual latent reasoning pipeline"
  - "align reasoning embeddings with vision encoder"
  - "build a CoT compression training pipeline"
---

# Render-of-Thought: Visual Latent Reasoning via Rendered Chain-of-Thought

This skill enables Claude to implement and apply the Render-of-Thought (RoT) framework from Wang et al. (2026), which compresses verbose textual Chain-of-Thought (CoT) reasoning into compact latent embeddings by rendering intermediate reasoning steps as images during training. A frozen vision encoder from an existing Vision-Language Model (VLM) serves as a semantic anchor, aligning latent embeddings with a visually-grounded space. At inference time, image rendering is eliminated entirely -- the model generates dense latent tokens that encode the full reasoning chain, then decodes only the final answer. This achieves 3-4x token compression and substantial inference speedup while keeping reasoning traceable through the training-time visual grounding.

## When to Use

- When building a CoT compression pipeline to reduce inference cost for math or logical reasoning tasks
- When the user wants to implement the two-stage RoT training loop (projection head alignment, then LM fine-tuning)
- When integrating a frozen vision encoder (e.g., from Qwen-VL or similar VLMs) as a supervision signal for latent reasoning
- When designing a text-to-image rendering module that converts reasoning steps into single-line images for training data
- When the user needs to add adaptive or fixed-budget stopping to a latent-token generation pipeline
- When evaluating token compression vs. accuracy tradeoffs on benchmarks like GSM8K, MATH-500, or SVAMP
- When retrofitting an existing VLM with a lightweight projection head to enable visual latent reasoning without pre-training from scratch

## Key Technique

**Core Insight: Visual Rendering as Training-Time Supervision.** Most latent reasoning methods (e.g., Coconut, RELAY) train models to produce hidden "thinking" tokens aligned only to final outcomes, making intermediate reasoning opaque. RoT takes a different path: it renders each textual CoT step as a single-line image, passes it through a frozen vision encoder to produce target embeddings, and trains a projection head to map the LLM's hidden states into that same visual embedding space. The vision encoder acts as a "semantic anchor" -- it provides rich, structured supervision for the latent space without requiring any new pre-training. Because the encoder is frozen and pre-trained, the visual embeddings already carry strong semantic structure, and the projection head simply learns to project into that space.

**Two-Stage Training.** In Stage 1, the LLM backbone is frozen and only the projection head (a two-layer MLP with SwiGLU activation) is trained. The loss combines an L2 alignment term (matching projected hidden states to vision encoder outputs) with a cross-entropy prediction term for the answer and a special `<|img_end|>` termination token. In Stage 2, the projection head is frozen and the LLM is fine-tuned (via LoRA or full fine-tuning) to generate latent embeddings that, when projected, align with the visual targets and produce correct final answers.

**Inference Without Images.** At test time, there is no rendering and no vision encoder. The LLM generates a fixed number of latent tokens (e.g., 32 for GSM8K, 64 for MATH), the frozen projection head maps them, and the model then decodes the textual answer. The visual grounding from training ensures these latent tokens encode coherent reasoning. Fixed token budgets outperform dynamic termination in practice due to the instability of predicting `<|img_end|>` over continuous latent representations.

## Step-by-Step Workflow

1. **Prepare the CoT dataset.** Collect question-CoT-answer triples in JSONL format: `{"id": N, "question": "...", "cot": "...", "answer": "..."}`. Source CoT annotations from teacher models (e.g., DeepSeek, GPT-4) or existing datasets like GSM8K-Aug.

2. **Implement the text-to-image renderer.** Build a rendering function that converts each CoT string into a single-line PNG image with fixed height (32px), dynamic width (computed from text length and font metrics), 4px padding, 20px font size, black text on white background. Single-line layout is critical -- it ensures vision encoder patches are extracted left-to-right, preserving sequential alignment with the text token order.

3. **Render all CoT strings to images.** Batch-render the full training set. Store rendered images alongside the JSONL data. Verify that no text is clipped by checking that rendered width accommodates the full string.

4. **Extract vision encoder targets.** Load the frozen vision encoder from a VLM (e.g., Qwen3-VL-4B-Instruct). Pass each rendered image through the encoder to produce target visual embeddings V = {v_1, ..., v_K}. Cache these embeddings to disk to avoid recomputation during training.

5. **Build the projection head.** Implement a two-layer MLP with SwiGLU activation and hidden dimension 4096. Input: LLM hidden states at the positions corresponding to latent reasoning tokens. Output: vectors in the same dimensionality as the vision encoder embeddings.

6. **Train Stage 1 (projection alignment).** Freeze the LLM backbone entirely. Train only the projection head using the combined loss: `L = L_pred + 10.0 * L_align`, where `L_align = (1/K) * sum(||v_hat_t - v_t||^2)` and `L_pred` is cross-entropy over the answer tokens and the `<|img_end|>` token. Use learning rate 2e-5, batch size 16, for 2 epochs.

7. **Train Stage 2 (LM fine-tuning).** Freeze the projection head. Fine-tune the LLM using LoRA (or full fine-tuning if compute allows) with only the prediction loss. The frozen projection head provides implicit visual-alignment constraints. Same hyperparameters as Stage 1.

8. **Configure the inference pipeline.** Set a fixed latent token budget (e.g., 32 tokens for grade-school math, 64 for competition math). At inference, the model generates that many latent tokens, the projection head maps them (though the output is discarded -- the LLM's internal representations carry the reasoning), then the model switches to generating the textual answer with temperature 0.0 and max 64 new tokens.

9. **Evaluate compression and accuracy.** Compare token counts against explicit CoT baselines on GSM8K, MATH-500, and SVAMP. Measure wall-clock inference time. Expect 3-4x token reduction with <2% accuracy degradation on math benchmarks.

10. **Iterate on the token budget.** If accuracy drops, increase the latent token count. If latency is still too high, decrease it. The budget is the primary lever for the compression-accuracy tradeoff.

## Concrete Examples

**Example 1: Building the Text-to-Image Renderer**

User: "I need a Python function that renders a CoT reasoning string into a single-line image for the RoT pipeline."

Approach:
1. Use PIL/Pillow to create images with fixed height 32px
2. Compute dynamic width from font metrics to avoid blank regions
3. Render black text on white background with 4px padding

Output:
```python
from PIL import Image, ImageDraw, ImageFont

def render_cot_to_image(cot_text: str, output_path: str) -> Image.Image:
    """Render a CoT string as a single-line image for RoT training."""
    font_size = 20
    padding = 4
    height = 32

    try:
        font = ImageFont.truetype("/usr/share/fonts/truetype/dejavu/DejaVuSans.ttf", font_size)
    except OSError:
        font = ImageFont.load_default()

    # Compute dynamic width from text extent
    bbox = font.getbbox(cot_text)
    text_width = bbox[2] - bbox[0]
    width = text_width + 2 * padding

    img = Image.new("RGB", (width, height), color="white")
    draw = ImageDraw.Draw(img)
    draw.text((padding, (height - font_size) // 2), cot_text, fill="black", font=font)
    img.save(output_path)
    return img
```

**Example 2: Implementing the Stage 1 Training Loss**

User: "How do I implement the combined alignment and prediction loss for RoT Stage 1?"

Approach:
1. Compute L2 alignment loss between projected hidden states and cached vision embeddings
2. Compute cross-entropy for answer tokens and the `<|img_end|>` termination token
3. Combine with lambda=10.0 weighting for alignment

Output:
```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class RoTStageLoss(nn.Module):
    def __init__(self, lambda_align: float = 10.0):
        super().__init__()
        self.lambda_align = lambda_align

    def forward(
        self,
        projected_embeddings: torch.Tensor,   # (B, K, D) from projection head
        target_vision_embeddings: torch.Tensor, # (B, K, D) from frozen vision encoder
        answer_logits: torch.Tensor,            # (B, seq_len, vocab_size)
        answer_labels: torch.Tensor,            # (B, seq_len)
    ) -> dict[str, torch.Tensor]:
        # L2 alignment loss over K latent positions
        align_loss = F.mse_loss(projected_embeddings, target_vision_embeddings)

        # Cross-entropy prediction loss (covers answer + <|img_end|> token)
        pred_loss = F.cross_entropy(
            answer_logits.view(-1, answer_logits.size(-1)),
            answer_labels.view(-1),
            ignore_index=-100,
        )

        total = pred_loss + self.lambda_align * align_loss
        return {"total": total, "pred": pred_loss, "align": align_loss}
```

**Example 3: Setting Up the Inference Pipeline**

User: "I have a trained RoT model. How do I run inference with fixed token budget?"

Approach:
1. Load the fine-tuned LLM and frozen projection head
2. Generate a fixed number of latent tokens (no rendering needed)
3. Switch to answer decoding after the latent budget is exhausted

Output:
```python
def rot_inference(
    model,
    projection_head,
    tokenizer,
    question: str,
    num_vision_tokens: int = 32,
    max_answer_tokens: int = 64,
) -> str:
    """Run RoT inference with fixed latent token budget."""
    # Encode the question
    prompt = f"Question: {question}\nLet me reason visually:\n"
    input_ids = tokenizer.encode(prompt, return_tensors="pt").to(model.device)

    # Phase 1: Generate latent reasoning tokens (fixed budget)
    with torch.no_grad():
        latent_output = model.generate(
            input_ids,
            max_new_tokens=num_vision_tokens,
            do_sample=False,
            output_hidden_states=True,
            # These tokens are latent -- not decoded to text
        )

    # Phase 2: Generate the textual answer
    with torch.no_grad():
        answer_output = model.generate(
            latent_output,
            max_new_tokens=max_answer_tokens,
            temperature=0.0,
            do_sample=False,
        )

    # Decode only the answer portion
    answer_ids = answer_output[0, latent_output.shape[1]:]
    return tokenizer.decode(answer_ids, skip_special_tokens=True)
```

## Best Practices

- **Do** use single-line rendering with dynamic width. Multi-line wrapping introduces spatial ambiguity that breaks the left-to-right patch alignment between vision encoder tokens and text token order.
- **Do** freeze the vision encoder throughout both training stages. The whole point is to use pre-trained visual semantics as a stable anchor -- fine-tuning it destroys the plug-and-play property.
- **Do** use fixed token budgets at inference rather than dynamic `<|img_end|>` termination. Fixed budgets are empirically more stable because predicting a discrete stop token over continuous latent representations is unreliable.
- **Do** cache vision encoder embeddings to disk after rendering. Re-encoding images every epoch wastes significant GPU time.
- **Avoid** using large lambda values (>10.0) for alignment loss. Over-weighting alignment collapses the latent space toward pure visual mimicry and degrades answer prediction quality.
- **Avoid** setting the latent token budget too low for complex tasks. 32 tokens works for grade-school math (GSM8K), but competition-level math (MATH-500) needs 64+ tokens to maintain accuracy.

## Error Handling

- **Rendered images are blank or clipped.** Verify font loading (fallback to PIL default if system fonts are unavailable). Check that dynamic width calculation accounts for the full text extent including descenders.
- **Alignment loss diverges in Stage 1.** Ensure vision encoder outputs and projection head outputs are in the same dimensionality. Check that target embeddings are properly normalized if the vision encoder produces L2-normalized outputs.
- **Accuracy drops sharply compared to explicit CoT.** Increase the latent token budget first. If that fails, check that Stage 1 converged (alignment loss should be <0.01) before starting Stage 2.
- **`<|img_end|>` token never predicted during dynamic termination.** This is a known instability. Switch to fixed token budgets. If dynamic termination is required, train with a higher weight on the termination token in the prediction loss.
- **Out-of-memory during vision encoding.** Batch the rendering and encoding steps. Vision encoders for VLMs like Qwen3-VL can consume significant memory on long CoT strings that produce wide images -- consider chunking very long CoT into segments.

## Limitations

- **Requires a VLM with a compatible vision encoder.** The framework is designed for models like Qwen3-VL that have an accessible frozen vision encoder. Pure LLMs without a vision component cannot use this approach directly.
- **Training data must include explicit CoT annotations.** You need full reasoning chains to render -- this rules out tasks where only question-answer pairs are available without intermediate steps.
- **Fixed token budgets are task-specific.** The optimal latent token count varies by domain (32 for GSM8K, 64 for MATH). New tasks require tuning this hyperparameter.
- **Single-line rendering limits CoT length.** Very long reasoning chains produce extremely wide images that may exceed vision encoder input limits. For CoT strings >2000 characters, consider summarizing or chunking.
- **Not a general-purpose reasoning accelerator.** RoT is validated on mathematical and logical reasoning benchmarks. Its effectiveness on open-ended, creative, or multi-hop reasoning tasks is unproven.
- **Two-stage training adds pipeline complexity.** Compared to standard fine-tuning, RoT requires rendering infrastructure, vision encoder inference, and careful stage-by-stage training coordination.

## Reference

Wang, Y., Li, S., Li, P., Yang, X., & Tang, Y. (2026). *Render-of-Thought: Rendering Textual Chain-of-Thought as Images for Visual Latent Reasoning.* arXiv:2601.14750v2. [https://arxiv.org/abs/2601.14750v2](https://arxiv.org/abs/2601.14750v2)

Key sections to study: Section 3 (method) for the two-stage training pipeline and alignment loss formulation; Section 4.3 for ablations on token budget, dynamic vs. fixed termination, and lambda weighting; Figure 2 for the full architecture diagram showing how rendered images, the frozen vision encoder, and the projection head interact during training vs. inference.

Code: [https://github.com/TencentBAC/RoT](https://github.com/TencentBAC/RoT)