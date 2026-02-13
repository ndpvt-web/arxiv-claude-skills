---
name: "universal-anti-forensics-attack-against"
description: "Implement ForgeryEraser-style adversarial attacks against AIGC/deepfake detectors by manipulating CLIP embeddings via multi-modal guidance loss. Use when: 'attack image forgery detector', 'anti-forensics perturbation CLIP', 'evade deepfake detection', 'adversarial attack on AIGC detector', 'test robustness of forgery detector', 'red-team image authenticity classifier'."
---

# ForgeryEraser: Universal Anti-Forensics Attack via Multi-Modal Guidance

This skill enables Claude to implement adversarial attacks against image forgery detectors that use Vision-Language Model (VLM) backbones like CLIP. Based on the ForgeryEraser framework (Li et al., 2026), the core insight is that most modern AIGC detectors share a frozen CLIP encoder as their feature backbone — so perturbing images in the upstream CLIP embedding space transfers universally to downstream detectors without needing access to any specific detector. Rather than optimizing against classifier logits, this approach uses text-derived semantic anchors ("real photograph" vs. "AI-generated image") to steer forged image embeddings toward authentic representations and away from forgery signatures.

## When to Use

- When the user wants to **red-team or stress-test** an image forgery/deepfake detector that uses a CLIP backbone (SIDA, AIDE, FakeVLM, LEGION, Effort, Forensics Adapter, or similar)
- When implementing **adversarial robustness benchmarks** for AIGC detection systems in a security research context
- When building a **perturbation pipeline** that must transfer across multiple unknown downstream detectors without white-box access
- When the user asks to "test if my forgery detector is robust to adversarial attacks" or "generate adversarial examples against CLIP-based classifiers"
- When evaluating whether an **explainable forensics model** can be fooled into producing authentic-looking explanations for forged images
- When the user needs to understand **why CLIP-backbone detectors are vulnerable** and how to design defenses against embedding-space manipulation

## Key Technique

**The shared-backbone vulnerability.** Modern AIGC detectors (SIDA, AIDE, FakeVLM, LEGION, etc.) freeze a pretrained CLIP ViT-L/14 encoder and train only lightweight classification heads on top. Because the CLIP encoder is publicly available, an attacker can compute gradients through it without any knowledge of the downstream detector. Perturbations that shift an image's CLIP embedding are inherited by every detector built on that backbone.

**Multi-modal guidance loss (MMG).** Instead of targeting a specific classifier's decision boundary, ForgeryEraser defines semantic anchors using CLIP's text encoder. Authentic anchors are text embeddings of prompts like "natural ISO noise" and "DSLR camera photograph"; forgery anchors encode prompts like "waxy skin texture" and "generative artifacts." The loss has two components: a **pull loss** that maximizes cosine similarity between the perturbed image embedding and authentic anchors, and a **push loss** that minimizes similarity with forgery anchors. The total loss is `L_MMG = Σ_l ω_l (L_pull^l + λ · L_push^l)` summed over the penultimate and final transformer blocks. This is optimized using Momentum Iterative FGSM (MI-FGSM) with differentiable anti-aliased resampling to suppress aliasing artifacts.

**Why it works so well.** By operating on semantic embeddings rather than pixel-level statistics, the perturbations are robust to JPEG compression (quality 50) and Gaussian blur (σ ≤ 3). The attack reduces detector accuracy from 90%+ to under 15% at ε = 8/255, and the semantic grounding means explainable forensic models are fooled into generating "authentic" explanations for forged images.

## Step-by-Step Workflow

1. **Identify the target detector's backbone.** Confirm the detector uses a CLIP-family encoder (ViT-L/14 is most common; some use ConvNeXt from OpenCLIP). Load the matching pretrained CLIP model — the attack does NOT require access to the detector's classification head.

2. **Select the attack scenario.** Choose between **global synthesis** (full AI-generated images from Stable Diffusion, Midjourney, etc.) or **local editing** (inpainting, face-swap, splicing). This determines which text prompts to use for anchor construction.

3. **Construct text-derived semantic anchors.** Encode scenario-specific text prompts through CLIP's text encoder and L2-normalize:
   - *Global synthesis authentic anchors*: "natural ISO noise", "DSLR camera photograph", "optical lens distortion", "authentic film grain"
   - *Global synthesis forgery anchors*: "waxy skin texture", "generative artifacts", "unnaturally smooth surface", "AI-generated pattern"
   - *Local editing authentic anchors*: "seamless blending", "consistent depth of field", "natural shadow continuity"
   - *Local editing forgery anchors*: "hard copy-paste edges", "unnatural boundary artifacts", "inconsistent lighting direction"

4. **Extract intermediate CLIP features.** Register forward hooks on the penultimate and final transformer blocks (layers N-1 and N) of the CLIP vision encoder. Pass the forged image through and collect feature maps at both layers.

5. **Compute the multi-modal guidance loss.** For each target layer `l` in {N-1, N}:
   - `L_pull^l = Σ_{a ∈ A_real} (1 - cos(z_adv^l, a))` — pull toward authentic anchors
   - `L_push^l = Σ_{a ∈ A_fake} cos(z_adv^l, a)` — push away from forgery anchors
   - `L_MMG = Σ_l (1/|S|) · (L_pull^l + λ · L_push^l)` with λ = 1.0

6. **Run MI-FGSM optimization.** Initialize perturbation δ = 0 and momentum g = 0. For T iterations:
   - Compute gradient: `∇_δ L_MMG`
   - Update momentum: `g_{t+1} = μ · g_t + ∇_δ L_MMG / ||∇_δ L_MMG||_1` (μ = 1.0)
   - Update perturbation: `δ_{t+1} = clip_ε(δ_t - α · sign(g_{t+1}))`
   - Standard budget: ε = 8/255, α = 2/255, T = 8; Low budget: ε = 4/255, α = 1/255, T = 4

7. **Apply differentiable resampling.** If the detector uses a different input resolution than the original image, apply anti-aliased interpolation during the optimization loop (not as a post-hoc step) to prevent aliasing artifacts from degrading transferability.

8. **Generate the adversarial image.** Add the final perturbation to the original forged image: `x_adv = clip_[0,1](x + δ)`. Save as PNG to avoid JPEG recompression artifacts during evaluation.

9. **Evaluate transferability.** Test the adversarial image against all available CLIP-backbone detectors. Measure accuracy drop, AUC degradation, and — for explainable models — whether the generated explanation now matches authentic image patterns.

10. **Iterate on anchor selection.** If attack success is low against a specific detector, refine text prompts to better target the forensic features that detector relies on. Matched anchors (global prompts for global detectors, local prompts for local detectors) significantly outperform mismatched ones.

## Concrete Examples

**Example 1: Red-teaming a CLIP-based deepfake detector**

User: "I have a deepfake detector that uses a frozen CLIP ViT-L/14 backbone with a linear probe. I want to test its adversarial robustness. Can you write the attack code?"

Approach:
1. Load OpenAI CLIP ViT-L/14 with `clip.load("ViT-L/14")`
2. Define authentic anchors: encode ["a photograph taken with a DSLR camera", "natural image with film grain and ISO noise"]
3. Define forgery anchors: encode ["an AI-generated face with waxy skin", "image with generative neural network artifacts"]
4. Register hooks on layers 22 and 23 (penultimate and final of ViT-L/14's 24 transformer blocks)
5. Run MI-FGSM for 8 steps with ε=8/255, α=2/255, μ=1.0, λ=1.0

Output:
```python
import torch
import clip
from PIL import Image
from torchvision import transforms

device = "cuda" if torch.cuda.is_available() else "cpu"
model, preprocess = clip.load("ViT-L/14", device=device)
model.eval()

# --- Anchor construction ---
authentic_prompts = [
    "a photograph taken with a DSLR camera",
    "natural image with film grain and ISO noise",
    "real photograph with optical lens distortion",
]
forgery_prompts = [
    "an AI-generated image with waxy skin texture",
    "image with generative neural network artifacts",
    "synthetically produced image with unnatural smoothness",
]

with torch.no_grad():
    auth_tokens = clip.tokenize(authentic_prompts).to(device)
    forg_tokens = clip.tokenize(forgery_prompts).to(device)
    auth_anchors = model.encode_text(auth_tokens)
    forg_anchors = model.encode_text(forg_tokens)
    auth_anchors = auth_anchors / auth_anchors.norm(dim=-1, keepdim=True)
    forg_anchors = forg_anchors / forg_anchors.norm(dim=-1, keepdim=True)

# --- Hook intermediate features ---
features = {}
def hook_fn(name):
    def hook(module, input, output):
        features[name] = output
    return hook

# ViT-L/14 has 24 transformer blocks in visual.transformer.resblocks
model.visual.transformer.resblocks[22].register_forward_hook(hook_fn("layer_22"))
model.visual.transformer.resblocks[23].register_forward_hook(hook_fn("layer_23"))

# --- MI-FGSM attack ---
def mmg_loss(feat_dict, auth_anchors, forg_anchors, lam=1.0):
    total = 0.0
    for name in feat_dict:
        # Use CLS token (index 0) from each layer's output
        z = feat_dict[name][:, 0, :]  # [B, D]
        z = z / z.norm(dim=-1, keepdim=True)
        pull = (1 - torch.cosine_similarity(
            z.unsqueeze(1), auth_anchors.unsqueeze(0), dim=-1
        )).sum(dim=-1).mean()
        push = torch.cosine_similarity(
            z.unsqueeze(1), forg_anchors.unsqueeze(0), dim=-1
        ).sum(dim=-1).mean()
        total += (pull + lam * push) / len(feat_dict)
    return total

def forgery_eraser_attack(image_tensor, eps=8/255, alpha=2/255,
                          steps=8, mu=1.0, lam=1.0):
    delta = torch.zeros_like(image_tensor, requires_grad=True)
    momentum = torch.zeros_like(image_tensor)

    for t in range(steps):
        adv = (image_tensor + delta).clamp(0, 1)
        # Forward through CLIP visual encoder
        features.clear()
        model.encode_image(adv)  # triggers hooks

        loss = mmg_loss(features, auth_anchors, forg_anchors, lam)
        loss.backward()

        grad = delta.grad.data
        grad_norm = grad / (grad.abs().mean(dim=[1,2,3], keepdim=True) + 1e-12)
        momentum = mu * momentum + grad_norm
        delta.data = delta.data - alpha * momentum.sign()
        delta.data = delta.data.clamp(-eps, eps)
        delta.data = (image_tensor + delta.data).clamp(0, 1) - image_tensor
        delta.grad.zero_()

    return (image_tensor + delta.data).clamp(0, 1)

# --- Run attack ---
img = preprocess(Image.open("forged_image.png")).unsqueeze(0).to(device)
adv_img = forgery_eraser_attack(img)
```

**Example 2: Benchmarking attack transferability across multiple detectors**

User: "I need to evaluate whether a single adversarial perturbation transfers across SIDA, AIDE, and FakeVLM detectors."

Approach:
1. Generate adversarial images using only the CLIP ViT-L/14 surrogate (no detector access)
2. Save perturbed images as lossless PNG
3. Run each detector independently on the same adversarial images
4. Compare accuracy before/after attack, report relative drop

Output:
```python
# After generating adv_images with forgery_eraser_attack():
from pathlib import Path
import json

results = {}
for detector_name, detector_fn in [
    ("SIDA", sida_predict), ("AIDE", aide_predict), ("FakeVLM", fakevlm_predict)
]:
    clean_acc = evaluate(detector_fn, clean_images, labels)
    adv_acc = evaluate(detector_fn, adv_images, labels)
    results[detector_name] = {
        "clean_accuracy": clean_acc,
        "adversarial_accuracy": adv_acc,
        "relative_drop_pct": round((clean_acc - adv_acc) / clean_acc * 100, 1),
    }

# Expected output (ε=8/255):
# {"SIDA":  {"clean_accuracy": 93.7, "adversarial_accuracy": 10.0, "relative_drop_pct": 89.3},
#  "AIDE":  {"clean_accuracy": 96.1, "adversarial_accuracy": 14.2, "relative_drop_pct": 85.2},
#  "FakeVLM": {"clean_accuracy": 99.3, "adversarial_accuracy": 37.6, "relative_drop_pct": 62.1}}
```

**Example 3: Choosing anchors for a local-editing attack scenario**

User: "I want to attack a detector that identifies inpainted regions. What prompts should I use?"

Approach:
1. Use local-editing-specific anchors targeting boundary artifacts
2. Adjust prompts to describe spatial consistency rather than global texture

Output:
```python
# Local editing anchors — target structural/boundary features
authentic_prompts = [
    "seamless image blending with consistent depth of field",
    "natural shadow continuity across the scene",
    "uniform lighting direction and color temperature",
    "coherent edge transitions between objects",
]
forgery_prompts = [
    "hard copy-paste edges with visible boundaries",
    "inconsistent lighting direction at splice boundary",
    "unnatural depth discontinuity between regions",
    "color temperature mismatch at edited region border",
]
# Encode and normalize as before, then run the same MI-FGSM loop.
# Matched local anchors outperform global anchors by ~15-25% relative
# drop on local editing detectors (per ablation in the paper).
```

## Best Practices

- **Do** match anchor type to attack scenario: use global-synthesis prompts for fully generated images and local-editing prompts for inpainting/splicing attacks. Mismatched anchors significantly reduce effectiveness.
- **Do** target both penultimate and final transformer layers (not just the final output embedding). Multi-layer optimization improves transferability across detectors with different head architectures.
- **Do** save adversarial images as lossless PNG. JPEG recompression can destroy the carefully crafted high-frequency perturbations before you even evaluate.
- **Do** use momentum (μ = 1.0) in MI-FGSM. Vanilla PGD without momentum produces perturbations that overfit to specific frequency bands and transfer poorly.
- **Avoid** using ε > 8/255 — larger budgets cause visible artifacts that defeat the purpose of an imperceptible attack. If 8/255 is insufficient, refine your text anchors rather than increasing the perturbation budget.
- **Avoid** applying the attack in pixel space after JPEG compression. The optimization loop must operate on the clean forged image; compression should only be tested during evaluation to measure robustness.

## Error Handling

| Problem | Cause | Fix |
|---------|-------|-----|
| Attack barely reduces accuracy | Detector uses a non-CLIP backbone (e.g., raw ResNet, EfficientNet) | Verify the detector's feature extractor; this attack only transfers through shared VLM backbones |
| `RuntimeError: grad` not computed | CLIP model set to `no_grad` mode or wrong tensor detached | Ensure `delta.requires_grad_(True)` and that `model.encode_image` is called without `torch.no_grad()` |
| Perturbation visually noticeable | ε too large or resampling aliases | Reduce ε to 4/255 and enable anti-aliased interpolation in the optimization loop |
| Features dict empty after forward | Hooks registered on wrong layers | ViT-L/14 has 24 blocks (indices 0-23); hook indices 22 and 23 for penultimate/final |
| Poor transfer to OpenCLIP ConvNeXt detector | Backbone mismatch between surrogate and target | Generate separate perturbations using an OpenCLIP ConvNeXt surrogate, or ensemble both backbones |
| Attack fails after JPEG compression | High-frequency perturbation destroyed | Use differentiable JPEG approximation inside the optimization loop to improve robustness |

## Limitations

- **CLIP-backbone dependency.** The attack assumes downstream detectors use a CLIP-family encoder. Detectors built on entirely different architectures (e.g., frequency-domain analyzers, pixel-level statistical tests) are not directly vulnerable to this approach.
- **Perturbation budget trade-off.** At ε = 4/255 (low budget), some detectors retain 30-50% accuracy. The attack is not a guaranteed bypass — it is a stress test that reveals architectural vulnerabilities.
- **Text anchor quality matters.** Poorly chosen prompts produce weak semantic guidance. There is no automatic prompt optimization; anchor selection requires understanding what forensic features the target domain relies on.
- **Defense potential.** Adversarial training, input purification, or ensemble methods using heterogeneous backbones (mixing CLIP with non-CLIP encoders) could mitigate this attack. The vulnerability is architectural, not fundamental.
- **Ethical scope.** This technique is strictly for authorized security research, red-teaming, and robustness evaluation. It should not be used to bypass forensic systems for deceptive purposes.

## Reference

**Paper:** Li, H., Peng, R., Luo, A., Tan, S., & Chen, C. (2026). *Universal Anti-forensics Attack against Image Forgery Detection via Multi-modal Guidance.* arXiv:2602.06530v1. [https://arxiv.org/abs/2602.06530v1](https://arxiv.org/abs/2602.06530v1)

**Key takeaway:** Section 3 (Methodology) details the MMG loss formulation and MI-FGSM optimization; Table 1 shows transferability results across six detectors; Table 3 contains the ablation study proving matched anchors outperform mismatched ones by large margins.