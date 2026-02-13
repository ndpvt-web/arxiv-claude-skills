---
name: "explainable-deepfake-detection-rl"
description: "Build explainable deepfake detection systems using RL-enhanced Self-Blended Images and Chain-of-Thought reasoning. Use when asked to: 'detect deepfakes with explanations', 'build interpretable face forgery detection', 'generate CoT annotations for deepfake training data', 'implement GRPO reward for vision-language forensics', 'create self-blended image augmentation pipeline', 'train an MLLM for explainable media forensics'."
---

# Explainable Deepfake Detection with RL-Enhanced Self-Blended Images

This skill enables Claude to help build deepfake detection systems that not only classify images as real or fake but also provide human-readable Chain-of-Thought (CoT) explanations of *why* an image is forged. The core technique uses Self-Blended Images (SBI) -- synthetic forgeries created by blending a face with augmented versions of itself -- to automatically generate CoT training annotations without expensive human labeling. A Group Relative Policy Optimization (GRPO) reinforcement learning loop then refines the multimodal LLM's reasoning quality, detection accuracy, and cross-domain generalization simultaneously.

## When to Use

- When the user wants to build a deepfake detector that explains its decisions in natural language, not just outputs a binary label
- When the user needs to generate large-scale training data for vision-language deepfake detection without manual textual annotation
- When implementing Self-Blended Image augmentation pipelines for face forgery research
- When designing RL reward functions (format, accuracy, CoT quality) for vision-language model fine-tuning
- When the user asks about cross-dataset generalization for deepfake detection (FF++, CelebDF, DFDC)
- When building a GRPO training loop that dynamically adjusts synthetic data generation parameters based on model performance
- When extracting token-level classification probabilities from a multimodal LLM's output logits

## Key Technique

**Self-Blended Images (SBI)** solve the annotation bottleneck. Instead of collecting real deepfakes and paying annotators to describe forgery artifacts, SBI creates synthetic fakes by blending a real face with a transformed copy of itself using a randomly generated mask. Because the pipeline controls every augmentation parameter (color shift in LAB space, brightness delta in YCrCb, saturation change in HSV, JPEG compression level, translation offset, sharpening kernel, and affine warp), it can automatically produce a lookup table mapping each parameter to a textual description of the resulting artifact. These key captions are then fed alongside the image pair into an MLLM (e.g., InternVL3) to produce full CoT annotations -- reasoning chains that walk through each artifact type, classify its severity (imperceptible <5, slight 5-10, obvious >10 on the respective metric), and conclude with a real/fake verdict.

**GRPO reinforcement learning** then improves the model beyond supervised fine-tuning. The training loop samples groups of reasoning trajectories for each image, scores them with three reward signals -- a **format reward** (does the output follow the required CoT structure with `<think>` and `<answer>` tags), an **accuracy reward** (is the real/fake classification correct, extracted via token probability comparison of `>Real<` vs `>Fake<` tokens), and a **CoT quality reward** (does the explanation identify specific forgery features rather than hallucinate or give generic responses, scored 0-5). Policy gradients computed from group-relative advantages update the model, and critically, historical reward values feed back into the data generation pipeline to dynamically adjust augmentation difficulty -- making forgeries harder when the model is performing well, and easier when it struggles. This feedback loop is what gives the method strong cross-dataset generalization without ever seeing real deepfakes from the target domain during training.

## Step-by-Step Workflow

1. **Set up the base MLLM**: Install a vision-language model that supports image+text input and text generation (InternVL3 is the paper's choice; alternatives include LLaVA, Qwen-VL). Ensure you can extract per-token logits from the output layer for the RL phase.

2. **Build the SBI augmentation pipeline**: Implement seven augmentation channels -- color (LAB Euclidean distance), brightness (YCrCb Y-channel delta), saturation (HSV S-channel delta), JPEG compression (quality factor), spatial translation (pixel offset), sharpening (kernel intensity), and affine transformation (rotation + scale + shear matrix). Each channel must accept a numeric intensity parameter and return both the augmented image and the measured artifact magnitude.

3. **Create the mask generator**: Implement random facial region mask generation -- use convex hull masks from facial landmarks (68-point or MediaPipe), with random erosion/dilation and Gaussian blur on edges to simulate realistic blending boundaries. The mask determines which region of the augmented face replaces the original.

4. **Construct the SBI blending function**: Given an original face image, a random mask, and a set of augmentation parameters, produce the blended fake image. Apply `blended = original * (1 - mask) + augmented * mask` with alpha blending at boundaries. Store the full parameter vector alongside each generated image.

5. **Build the caption lookup table**: Define a mapping from each augmentation type and severity level to descriptive text fragments. For example: `{"color": {"imperceptible": "no noticeable color discrepancy", "slight": "subtle color shift near blending boundary", "obvious": "clear color mismatch between facial regions"}}`. Classify each parameter's measured magnitude against thresholds (e.g., LAB distance: <5 imperceptible, 5-10 slight, >10 obvious).

6. **Generate CoT annotations automatically**: For each SBI pair (real, fake), collect the key captions from the lookup table, format them as a structured prompt (e.g., "This image shows: [captions]. Provide a chain-of-thought analysis of whether this face is real or manipulated, referencing specific artifacts."), and feed the prompt + image pair into the MLLM to produce the initial CoT text. Store as `(image, CoT_text, real/fake_label)` triples.

7. **Implement the three-part reward function**:
   - **Format reward** (`r_format`): Parse the model output for required structural tags (`<think>...</think>`, `<answer>Real|Fake</answer>`). Return 1.0 if valid, 0.0 otherwise.
   - **Accuracy reward** (`r_acc`): Extract logits at the answer token position, compute softmax over the `Real` and `Fake` token IDs, return 1.0 if argmax matches ground truth, 0.0 otherwise.
   - **CoT quality reward** (`r_cot`): Score the reasoning on a 0-5 rubric -- 0 for wrong classification, 1-2 for hallucinated/generic explanations, 3-4 for identifying one to three specific artifacts, 5 for comprehensive analysis. This can be automated with a separate LLM judge or a learned reward model.

8. **Run GRPO training**: For each training image, sample G candidate reasoning trajectories from the current policy. Score each with the composite reward `r = w1*r_format + w2*r_acc + w3*r_cot`. Compute group-relative advantages (subtract mean reward within the group, divide by standard deviation). Update the policy with clipped surrogate objective and KL divergence penalty against the reference model.

9. **Implement the feedback-driven data loop**: After each GRPO epoch, compute aggregate reward statistics per augmentation difficulty level. If mean reward for a difficulty tier exceeds a threshold, shift the SBI generation distribution toward harder augmentations (smaller artifact magnitudes, closer to imperceptible). If reward drops, shift back. Re-generate a portion of the training data with updated parameters before the next epoch.

10. **Evaluate with cross-dataset protocol**: Test on unseen deepfake datasets (FF++, CelebDF, DFDC, Google DFD) without fine-tuning. Report both detection metrics (AUC, accuracy, F1) and explanation quality (human evaluation on 0-5 scale, or GPT-based scoring). Compare per-domain to assess generalization.

## Concrete Examples

**Example 1: Building the SBI augmentation pipeline in Python**

User: "I need to create a Self-Blended Image generation pipeline for deepfake detection training data."

Approach:
1. Implement face detection and landmark extraction (dlib or MediaPipe)
2. Build the seven augmentation channels with parameterized intensity
3. Generate random convex-hull masks from landmarks with boundary smoothing
4. Blend augmented face back onto original and record artifact measurements

Output:
```python
import cv2
import numpy as np
from dataclasses import dataclass

@dataclass
class AugmentationParams:
    color_shift: float = 0.0       # LAB Euclidean distance
    brightness_delta: float = 0.0  # YCrCb Y-channel delta
    saturation_delta: float = 0.0  # HSV S-channel delta
    jpeg_quality: int = 100        # JPEG compression quality (lower = more artifacts)
    translation_px: float = 0.0    # pixel offset
    sharpen_intensity: float = 0.0 # sharpening kernel strength
    affine_angle: float = 0.0      # rotation degrees

THRESHOLDS = {
    "color_shift":       (5.0, 10.0),
    "brightness_delta":  (5.0, 10.0),
    "saturation_delta":  (5.0, 10.0),
    "jpeg_quality":      (80, 60),   # inverted: lower quality = more obvious
    "translation_px":    (2.0, 5.0),
    "sharpen_intensity": (0.3, 0.7),
    "affine_angle":      (2.0, 5.0),
}

def classify_severity(param_name: str, value: float) -> str:
    low, high = THRESHOLDS[param_name]
    if param_name == "jpeg_quality":
        if value > low: return "imperceptible"
        if value > high: return "slight"
        return "obvious"
    if abs(value) < low: return "imperceptible"
    if abs(value) < high: return "slight"
    return "obvious"

def generate_sbi(image: np.ndarray, landmarks: np.ndarray,
                 params: AugmentationParams) -> tuple[np.ndarray, dict]:
    """Generate a Self-Blended Image and artifact severity map."""
    # 1. Create convex hull mask from facial landmarks
    hull = cv2.convexHull(landmarks)
    mask = np.zeros(image.shape[:2], dtype=np.float32)
    cv2.fillConvexPoly(mask, hull, 1.0)
    mask = cv2.GaussianBlur(mask, (15, 15), 5)

    # 2. Apply augmentations to a copy
    augmented = image.copy()
    augmented = apply_color_shift(augmented, params.color_shift)
    augmented = apply_brightness(augmented, params.brightness_delta)
    augmented = apply_saturation(augmented, params.saturation_delta)
    augmented = apply_jpeg_compression(augmented, params.jpeg_quality)
    augmented = apply_translation(augmented, params.translation_px)
    augmented = apply_sharpening(augmented, params.sharpen_intensity)
    augmented = apply_affine(augmented, params.affine_angle)

    # 3. Alpha-blend
    mask_3ch = np.stack([mask] * 3, axis=-1)
    blended = (image * (1 - mask_3ch) + augmented * mask_3ch).astype(np.uint8)

    # 4. Measure actual artifact magnitudes and classify severity
    severities = {
        "color": classify_severity("color_shift", measure_lab_distance(image, blended, mask)),
        "brightness": classify_severity("brightness_delta", params.brightness_delta),
        "saturation": classify_severity("saturation_delta", params.saturation_delta),
        "compression": classify_severity("jpeg_quality", params.jpeg_quality),
        "translation": classify_severity("translation_px", params.translation_px),
        "sharpening": classify_severity("sharpen_intensity", params.sharpen_intensity),
        "affine": classify_severity("affine_angle", params.affine_angle),
    }
    return blended, severities
```

**Example 2: Implementing the GRPO reward function**

User: "How do I implement the three-part reward function for RL-based deepfake detection training?"

Approach:
1. Parse model output for structural format compliance
2. Extract token probabilities for real/fake classification
3. Score CoT quality against rubric

Output:
```python
import re
import torch
import torch.nn.functional as F

def compute_format_reward(output_text: str) -> float:
    """Check that output follows <think>...</think><answer>Real|Fake</answer> format."""
    pattern = r"<think>.+?</think>\s*<answer>(Real|Fake)</answer>"
    return 1.0 if re.search(pattern, output_text, re.DOTALL) else 0.0

def compute_accuracy_reward(logits: torch.Tensor, tokenizer,
                            answer_position: int, ground_truth: str) -> float:
    """Extract P(Real) vs P(Fake) from logits at the answer token position."""
    probs = F.softmax(logits[answer_position], dim=-1)
    real_id = tokenizer.encode("Real", add_special_tokens=False)[0]
    fake_id = tokenizer.encode("Fake", add_special_tokens=False)[0]
    predicted = "Real" if probs[real_id] > probs[fake_id] else "Fake"
    return 1.0 if predicted == ground_truth else 0.0

def compute_cot_quality_reward(output_text: str, severities: dict) -> float:
    """Score CoT on 0-5 rubric based on specific artifact identification."""
    think_match = re.search(r"<think>(.*?)</think>", output_text, re.DOTALL)
    if not think_match:
        return 0.0
    reasoning = think_match.group(1).lower()

    # Count how many actual non-imperceptible artifacts are mentioned
    artifact_keywords = {
        "color": ["color", "hue", "tint", "chromatic"],
        "brightness": ["bright", "luminan", "exposure", "lighting"],
        "saturation": ["saturat", "vivid", "faded"],
        "compression": ["compress", "jpeg", "block", "artifact"],
        "translation": ["shift", "translat", "misalign", "offset"],
        "sharpening": ["sharp", "blur", "edge", "smooth"],
        "affine": ["rotat", "warp", "skew", "distort"],
    }
    visible = [k for k, v in severities.items() if v != "imperceptible"]
    mentioned = 0
    for artifact in visible:
        if any(kw in reasoning for kw in artifact_keywords[artifact]):
            mentioned += 1

    if not visible:  # real image
        return 4.0 if "no" in reasoning and "artifact" in reasoning else 2.0
    ratio = mentioned / len(visible)
    if ratio >= 0.8: return 5.0
    if ratio >= 0.5: return 4.0
    if ratio >= 0.3: return 3.0
    if mentioned > 0: return 2.0
    return 1.0

def composite_reward(output_text, logits, tokenizer, answer_pos,
                     ground_truth, severities,
                     w_format=0.1, w_acc=0.5, w_cot=0.4) -> float:
    r_fmt = compute_format_reward(output_text)
    r_acc = compute_accuracy_reward(logits, tokenizer, answer_pos, ground_truth)
    r_cot = compute_cot_quality_reward(output_text, severities)
    return w_format * r_fmt + w_acc * r_acc + w_cot * (r_cot / 5.0)
```

**Example 3: Feedback-driven augmentation difficulty adjustment**

User: "How do I dynamically adjust SBI generation difficulty based on training rewards?"

Approach:
1. Track per-difficulty-tier reward statistics across GRPO epochs
2. Shift sampling distribution toward harder augmentations when performance is high
3. Re-generate a fraction of training data each epoch

Output:
```python
import numpy as np
from collections import defaultdict

class FeedbackScheduler:
    """Adjusts SBI augmentation difficulty based on historical GRPO rewards."""

    def __init__(self, difficulty_tiers=("easy", "medium", "hard"),
                 initial_probs=(0.2, 0.5, 0.3), shift_rate=0.05,
                 reward_threshold=0.75):
        self.tiers = difficulty_tiers
        self.probs = np.array(initial_probs, dtype=np.float64)
        self.shift_rate = shift_rate
        self.threshold = reward_threshold
        self.reward_history = defaultdict(list)

    # Difficulty tiers map to augmentation magnitude ranges
    TIER_RANGES = {
        "easy":   (10.0, 20.0),   # obvious artifacts
        "medium": (5.0, 10.0),    # slight artifacts
        "hard":   (0.5, 5.0),     # near-imperceptible
    }

    def record_rewards(self, tier: str, rewards: list[float]):
        self.reward_history[tier].extend(rewards)

    def update_distribution(self):
        for i, tier in enumerate(self.tiers):
            if not self.reward_history[tier]:
                continue
            mean_r = np.mean(self.reward_history[tier][-200:])
            if mean_r > self.threshold and i < len(self.tiers) - 1:
                # Model is strong here -- shift probability toward harder tier
                shift = min(self.shift_rate, self.probs[i] * 0.5)
                self.probs[i] -= shift
                self.probs[i + 1] += shift
            elif mean_r < self.threshold * 0.6 and i > 0:
                # Model is struggling -- shift back toward easier tier
                shift = min(self.shift_rate, self.probs[i] * 0.5)
                self.probs[i] -= shift
                self.probs[i - 1] += shift
        self.probs = np.clip(self.probs, 0.05, 0.9)
        self.probs /= self.probs.sum()

    def sample_tier(self) -> str:
        return np.random.choice(self.tiers, p=self.probs)

    def sample_params(self) -> float:
        tier = self.sample_tier()
        lo, hi = self.TIER_RANGES[tier]
        return np.random.uniform(lo, hi), tier
```

## Best Practices

- **Do:** Measure artifact magnitudes in perceptual color spaces (LAB for color, YCrCb for brightness, HSV for saturation) rather than raw RGB pixel differences. This matches human perception of forgery artifacts.
- **Do:** Apply Gaussian blur to mask boundaries before blending. Sharp mask edges create unrealistically obvious forgeries that teach the model shortcuts instead of genuine forensic reasoning.
- **Do:** Weight the accuracy reward higher than CoT quality reward early in training (e.g., 0.6/0.3), then shift toward CoT quality later (e.g., 0.4/0.5) once classification stabilizes.
- **Do:** Validate CoT quality with both automated LLM-judge scoring and periodic human evaluation on a held-out set. The paper found human scores averaged 3.7/5 vs. GPT-based scores of 2.5/5, indicating automated scoring is conservative.
- **Avoid:** Training exclusively on high-severity (obvious) augmentations. The feedback scheduler exists because models overfit to easy artifacts and fail on subtle manipulations in real deepfakes.
- **Avoid:** Using the same MLLM for both CoT generation and CoT quality reward scoring. This creates a self-reinforcing bias. Use a separate model or a rule-based rubric for the quality reward.

## Error Handling

- **Face detection failure**: Skip images where no face or landmarks are detected. Do not generate SBIs from non-face images -- they produce meaningless training signal.
- **Degenerate masks**: If the convex hull mask covers <5% or >95% of the image, discard it. Extremely small masks produce imperceptible fakes regardless of augmentation; full-face masks eliminate the blending boundary signal entirely.
- **Format reward always 0**: If the model consistently fails format compliance, freeze RL and run a few epochs of supervised fine-tuning on correctly-formatted examples to establish the output structure before resuming GRPO.
- **Reward collapse (all samples score identically)**: Increase the GRPO group size G or widen the augmentation difficulty range to ensure variance in the reward distribution. Without variance, group-relative advantages are zero and no learning occurs.
- **Token ID mismatch**: Different tokenizers split "Real" and "Fake" into different subtoken sequences. Verify that your accuracy reward extracts logits at the correct position and for the correct token ID. Use `tokenizer.encode("Real")` and check it produces a single token; if not, use the first subtoken or switch to a tokenizer-appropriate answer format.

## Limitations

- **Requires a capable base MLLM**: The technique builds on top of models like InternVL3 that already understand images and can generate coherent text. It will not work with vision-only classifiers or small language models that cannot produce multi-sentence reasoning.
- **SBI =/= real deepfakes**: Self-blended images approximate forgery artifacts but do not replicate method-specific signatures (e.g., GAN frequency artifacts, diffusion model texture patterns). Cross-domain performance is competitive but not guaranteed against novel generation methods.
- **Compute-intensive RL loop**: GRPO requires sampling multiple trajectories per image per training step. Expect 3-5x the compute cost of standard supervised fine-tuning. The feedback-driven data regeneration adds further overhead.
- **CoT quality scoring is subjective**: Both human and automated evaluations of explanation quality are noisy. The 0-5 rubric requires calibration to the specific domain, and reward signal quality directly affects what the model learns to explain.
- **Binary classification only**: The framework detects real vs. fake and explains artifacts but does not identify the specific deepfake generation method (e.g., FaceSwap vs. diffusion). Attribution requires additional training signal.

## Reference

- **Paper**: [Explainable Deepfake Detection with RL Enhanced Self-Blended Images](https://arxiv.org/abs/2601.15624v1) (ICASSP 2026) -- Focus on Section 3 for the SBI-based CoT data pipeline, Section 4 for the GRPO reward design and feedback scheduler, and Table 1-2 for cross-dataset benchmark results.
- **Code**: [github.com/deon1219/rlsbi](https://github.com/deon1219/rlsbi) -- Reference implementation with token probability extraction and augmentation parameter definitions.