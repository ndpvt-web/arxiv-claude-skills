---
name: "veq-modality-adaptive-quantization-moe"
description: "Apply VEQ modality-adaptive quantization to compress MoE Vision-Language Models with minimal accuracy loss. Implements dual-aware PTQ that respects cross-modal differences and expert heterogeneity. Use when: 'quantize Kimi-VL to 3-bit', 'compress MoE VLM for deployment', 'apply VEQ quantization to Qwen3-VL', 'reduce MoE vision-language model memory', 'set up modality-aware PTQ pipeline', 'optimize expert quantization for multimodal model'."
---

# VEQ: Modality-Adaptive Quantization for MoE Vision-Language Models

This skill enables Claude to apply Visual Expert Quantization (VEQ), a post-training quantization framework that compresses Mixture-of-Experts Vision-Language Models (MoE VLMs) like Kimi-VL and Qwen3-VL to 3-bit or 4-bit weights while preserving multimodal accuracy. VEQ is unique because it accounts for two forms of heterogeneity that generic quantization ignores: the asymmetric importance of vision vs. language tokens, and the uneven contribution of different experts in the MoE architecture. Under W3A16 (3-bit weights, 16-bit activations), VEQ achieves +2.04% accuracy on Kimi-VL and +3.09% on Qwen3-VL over prior SOTA quantization methods.

## When to Use

- When the user needs to deploy an MoE VLM (Kimi-VL, Qwen3-VL, or similar architectures) under tight GPU memory constraints
- When applying standard PTQ methods (RTN, AWQ, GPTQ) to an MoE VLM and seeing disproportionate accuracy drops on vision-heavy benchmarks
- When building a quantization pipeline that must handle both vision and language tokens through a shared MoE backbone
- When the user asks to compress a multimodal model and wants to preserve performance on tasks like VQA, OCR, diagram understanding, or science QA
- When configuring expert-level bit allocation or calibration weighting for MoE models
- When setting up SGLang or vLLM inference for quantized MoE VLMs and selecting the right quantization strategy

## Key Technique

**The core insight:** In MoE VLMs, text tokens have ~22.4x higher gradient magnitudes than vision tokens despite being less frequent, and a small fraction of experts disproportionately determines output quality. Standard quantization treats all tokens and all experts equally, which wastes precision budget on low-impact experts and misweights modality contributions during calibration.

**VEQ-ME (Modality-Expert-Aware Quantization)** computes an importance score for each expert based on how many text and vision tokens route through it, normalized by modality gradient sensitivity. The formula is `S_i = gamma * N_i_text + beta * N_i_vis`, where `beta = T_text / T_vis` corrects for token count imbalance and `gamma = ||grad_text|| / ||grad_vis||` corrects for gradient magnitude asymmetry. The quantization loss function is then weighted: `L = sum(S_i * ||W_i*X_i - W_hat_i*X_i||^2)`, ensuring high-activation, gradient-sensitive experts get tighter reconstruction. This modifies AWQ-style channel scaling.

**VEQ-MA (Modality-Affinity-Aware Quantization)** enhances the Hessian matrix used in GPTQ-style quantization. Instead of `H = X*X^T`, it uses `H_tilde = X*C*X^T` where the diagonal matrix C weights each calibration token by `c_j = p_j * alpha_j` -- the product of router confidence (`p_j`, how strongly the router assigns that token to this expert) and modality weight (`alpha_j = gamma` for text, `1` for vision). This means the Hessian-guided weight update prioritizes columns that matter most for high-affinity, high-gradient tokens.

## Step-by-Step Workflow

1. **Identify the target MoE VLM architecture.** Confirm the model uses Mixture-of-Experts layers (e.g., Kimi-VL-Instruct, Qwen3-VL-30B-A3B). Check that MoE is in the language model backbone, not the vision encoder, since VEQ targets the MoE FFN layers specifically.

2. **Prepare a multimodal calibration dataset.** Collect 128-256 samples containing both image-text and text-only inputs representative of your deployment distribution. Use frameworks like `lmms-eval` to format calibration data. The dataset must include both modalities -- text-only calibration will not expose the cross-modal heterogeneity VEQ exploits.

3. **Run a calibration forward pass to collect expert activation statistics.** For each calibration sample, record per-expert token counts split by modality: `N_i_text` and `N_i_vis` for each expert `i`. Also record router logits/probabilities `p_j` for each token-expert assignment. Store gradient norms aggregated by modality to compute `gamma = ||grad_text|| / ||grad_vis||`.

4. **Compute expert importance scores (VEQ-ME).** Calculate the normalization factor `beta = T_text / T_vis` (total text tokens / total vision tokens) and gradient ratio `gamma`. For each expert, compute `S_i = gamma * N_i_text + beta * N_i_vis`. Normalize scores so they sum to the number of experts.

5. **Apply VEQ-ME: weighted channel scaling (AWQ-style).** Modify the AWQ quantization pass to use the importance scores `S_i` as per-expert loss weights. When searching for optimal per-channel scaling factors, minimize `S_i * ||W_i*X_i - Q(s*W_i)*X_i/s||^2` instead of the unweighted error. Experts with higher `S_i` get tighter quantization.

6. **Construct the enhanced Hessian (VEQ-MA).** For each expert's weight matrix, build the modified Hessian `H_tilde = X * diag(c_1, ..., c_n) * X^T` where `c_j = p_j * alpha_j`. Use `alpha_j = gamma` for text tokens and `alpha_j = 1.0` for vision tokens. This replaces the standard `X*X^T` Hessian in GPTQ.

7. **Apply VEQ-MA: Hessian-guided weight quantization (GPTQ-style).** Run the GPTQ column-by-column quantization using `H_tilde` instead of the standard Hessian. The modified Hessian steers quantization error away from weight dimensions that high-affinity, high-gradient tokens rely on.

8. **Select the target bit-width and pack weights.** Choose W3A16 (3-bit weights, 16-bit activations) for maximum compression (~5x reduction) or W4A16 (4-bit) for a safer accuracy-memory tradeoff. Pack quantized weights using group quantization (group size 128 is standard).

9. **Validate on multimodal benchmarks.** Evaluate the quantized model on diverse tasks: MMMU (reasoning), MMBench (general), TextVQA/InfoVQA (OCR), AI2D/ScienceQA (diagrams), VizWiz-VQA (accessibility). Compare against RTN, AWQ, and GPTQ baselines to confirm VEQ's improvement.

10. **Deploy with an MoE-optimized inference backend.** Use SGLang or vLLM with MoE kernel support to serve the quantized model. Ensure the runtime supports the chosen weight format (e.g., GPTQ-style packed 3-bit integers with group quantization).

## Concrete Examples

**Example 1: Quantize Qwen3-VL-30B to 3-bit for single-GPU deployment**

User: "I want to run Qwen3-VL-30B-A3B on a single 24GB GPU. Can you set up VEQ quantization?"

Approach:
1. Verify Qwen3-VL-30B-A3B has MoE in its LLM backbone with ~3B active parameters out of 30B total
2. Set up calibration using lmms-eval with 128 multimodal samples from a mixed VQA/captioning dataset
3. Collect activation stats: text tokens show ~22x higher gradient norms than vision tokens; 8 out of 64 experts handle >40% of total routing
4. Compute importance scores with gamma ≈ 22.4, beta from token ratio
5. Apply VEQ-ME weighted AWQ scaling, then VEQ-MA enhanced-Hessian GPTQ at W3G128

Output:
```python
# veq_quantize.py - VEQ quantization config for Qwen3-VL-30B-A3B
import torch
from transformers import AutoModelForCausalLM, AutoTokenizer

model_id = "Qwen/Qwen3-VL-30B-A3B-Instruct"

# Step 1: Load model and collect calibration statistics
veq_config = {
    "model_id": model_id,
    "bits": 3,                    # W3A16
    "group_size": 128,            # G128 group quantization
    "calibration_samples": 128,
    "calibration_dataset": "lmms-eval-mixed-vqa",  # must include images + text

    # VEQ-ME: Expert importance weighting
    "expert_weighting": {
        "enabled": True,
        "gamma_source": "gradient_norm_ratio",  # ||grad_text|| / ||grad_vis||
        "beta_source": "token_count_ratio",     # T_text / T_vis
    },

    # VEQ-MA: Enhanced Hessian construction
    "hessian_enhancement": {
        "enabled": True,
        "use_router_affinity": True,   # weight by p_j (router confidence)
        "modality_weighting": True,    # alpha_j = gamma for text, 1 for vision
    },

    # Quantization method
    "method": "gptq",  # VEQ-MA builds on GPTQ's Hessian framework
    "awq_prescale": True,  # VEQ-ME applies AWQ-style scaling first
}

# Expected result: ~6GB model (down from ~30GB fp16 total, ~3GB active)
# Accuracy: ~62% MMMU, ~78% TextVQA (vs ~59% MMMU, ~74% TextVQA with vanilla GPTQ)
```

**Example 2: Diagnose accuracy drop after naive quantization of Kimi-VL**

User: "I quantized Kimi-VL with standard GPTQ to 3-bit and accuracy dropped 8% on TextVQA but only 2% on ScienceQA. What's going on?"

Approach:
1. TextVQA is OCR-heavy (relies on vision tokens); ScienceQA is more reasoning-heavy (relies on text tokens)
2. Standard GPTQ uses uniform Hessian -- vision-critical experts get same treatment as text-critical ones
3. In MoE VLMs, certain experts specialize in vision tokens but have lower activation frequency, so uniform quantization under-serves them
4. Apply VEQ-MA to upweight vision-token-routed experts in the Hessian calibration

Output:
```
Diagnosis: Standard GPTQ's Hessian (H = X*X^T) is dominated by text tokens because
they have higher gradient magnitudes. Vision-specialist experts get quantized with
insufficient precision, causing disproportionate OCR/vision accuracy loss.

Fix with VEQ:
1. Run calibration with modality-tagged tokens
2. Compute router affinity scores p_j per token-expert pair
3. Build enhanced Hessian: H_tilde = X * diag(p_j * alpha_j) * X^T
   - For vision tokens: alpha_j = 1.0 (base weight)
   - For text tokens: alpha_j = gamma (~22.4, reflecting gradient sensitivity)
4. This rebalances the Hessian so vision-specialist experts get appropriate
   quantization precision proportional to their actual output impact

Expected improvement: +2-3% on TextVQA while maintaining ScienceQA performance.
```

**Example 3: Implement expert importance scoring for a custom MoE model**

User: "I have a custom MoE VLM with 32 experts. How do I compute VEQ importance scores?"

Output:
```python
import torch

def compute_veq_expert_scores(
    expert_text_counts: torch.Tensor,   # shape: [num_experts], tokens routed per expert (text)
    expert_vis_counts: torch.Tensor,    # shape: [num_experts], tokens routed per expert (vision)
    text_grad_norm: float,              # aggregated gradient norm for text tokens
    vis_grad_norm: float,               # aggregated gradient norm for vision tokens
) -> torch.Tensor:
    """
    Compute VEQ-ME expert importance scores.

    Returns normalized per-expert importance weights for quantization loss.
    """
    num_experts = expert_text_counts.shape[0]

    # Gradient sensitivity ratio: text tokens matter more per-token
    gamma = text_grad_norm / (vis_grad_norm + 1e-8)

    # Token count normalization: correct for imbalanced modality volumes
    T_text = expert_text_counts.sum()
    T_vis = expert_vis_counts.sum()
    beta = T_text / (T_vis + 1e-8)

    # Expert importance score (Eq. from VEQ paper)
    S = gamma * expert_text_counts.float() + beta * expert_vis_counts.float()

    # Normalize so scores average to 1.0
    S = S / (S.mean() + 1e-8)

    return S  # shape: [num_experts]

# Usage during quantization:
# For each expert i, multiply quantization loss by S[i]:
# loss_i = S[i] * ||W_i @ X_i - Q(W_i) @ X_i||^2_F
```

## Best Practices

- **Do:** Always calibrate with mixed-modality data. Text-only calibration completely misses the vision-expert heterogeneity that VEQ exploits. Use at least 30% image-containing samples.
- **Do:** Compute gamma empirically from your specific model rather than assuming a fixed ratio. The 22.4x gradient ratio was measured on specific models and varies by architecture.
- **Do:** Apply VEQ-ME (AWQ-style scaling) before VEQ-MA (GPTQ-style Hessian quantization). The scaling normalization from VEQ-ME improves the conditioning for the subsequent Hessian-guided pass.
- **Do:** Validate on both vision-heavy (TextVQA, InfoVQA) and reasoning-heavy (MMMU, ScienceQA) benchmarks to confirm balanced improvement.
- **Avoid:** Applying VEQ to the vision encoder itself. VEQ targets MoE layers in the language model backbone -- the vision encoder (typically a dense ViT) should use standard quantization.
- **Avoid:** Using VEQ on dense (non-MoE) VLMs. The expert heterogeneity component is meaningless without MoE routing. For dense VLMs, standard AWQ/GPTQ suffices.

## Error Handling

- **Router statistics are uniform across experts:** If all experts show near-equal activation counts, the model may use auxiliary load-balancing loss that flattens routing. VEQ-ME scores will converge to uniform weights, reducing to standard quantization. Check if the model was trained with strong load balancing -- VEQ benefits most from models with natural routing imbalance.
- **Gradient norm computation is unstable:** For very large models, computing full gradient norms over calibration data can cause OOM. Use gradient accumulation over micro-batches or approximate with a subset of layers (the last 25% of MoE layers contribute most to gradient variance).
- **Quantized model produces garbled vision outputs:** The enhanced Hessian may have numerical issues if router affinities `p_j` are extremely concentrated (near 0 or 1). Add a small smoothing term: `c_j = max(p_j, 0.01) * alpha_j`.
- **Calibration dataset mismatch:** If calibration data distribution differs significantly from deployment (e.g., calibrating on natural images but deploying on documents), the expert activation statistics will be misleading. Match calibration to expected deployment distribution.

## Limitations

- VEQ is designed specifically for MoE-architecture VLMs. It provides no benefit for dense VLMs (LLaVA, InternVL) or text-only MoE models (Mixtral without vision).
- The method requires access to the router's token-expert assignment probabilities during calibration, which not all inference frameworks expose by default.
- VEQ has been validated primarily on Kimi-VL and Qwen3-VL. Transferability to other MoE VLMs (e.g., future architectures with different routing mechanisms like expert choice) needs verification.
- At W2 (2-bit) quantization, all PTQ methods including VEQ degrade significantly. VEQ is most effective at W3 and W4 bit-widths.
- The gradient norm computation for gamma adds calibration overhead compared to vanilla AWQ/GPTQ (roughly 2x calibration time, but still no training required).

## Reference

**Paper:** [VEQ: Modality-Adaptive Quantization for MoE Vision-Language Models](https://arxiv.org/abs/2602.01037v1) (Qin et al., 2026)
**Code:** [github.com/guangshuoqin/VEQ](https://github.com/guangshuoqin/VEQ)
**Key insight to look for:** Section 3's analysis of cross-modal gradient asymmetry (text tokens having ~22.4x higher gradient magnitudes) and the derivation of the enhanced Hessian matrix H_tilde = X*C*X^T that integrates router affinity with modality weighting.