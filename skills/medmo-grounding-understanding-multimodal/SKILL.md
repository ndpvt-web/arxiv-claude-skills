---
name: "medmo-grounding-understanding-multimodal"
description: "Build medical image analysis pipelines with multi-stage grounded reasoning: cross-modal alignment, instruction-tuned VQA/report generation, and RL-based spatial grounding with bounding boxes. Use when asked to 'build a medical image grounding system', 'create a radiology VQA pipeline', 'implement grounded disease localization', 'set up medical report generation from images', 'design a multi-modal medical AI training pipeline', or 'add bounding box grounding to a medical MLLM'."
---

# MedMO: Grounded Medical Image Understanding with Multi-Stage Training

This skill enables Claude to help users build medical image analysis systems that combine visual question answering, report generation, and spatially-grounded disease localization using the MedMO architecture. The core technique is a four-stage training pipeline (general medical SFT, high-resolution grounding, instruction tuning, GRPO reinforcement learning with verifiable rewards) applied to a vision-language model, where bounding-box grounding is enforced through a GIoU-based reward signal during RL. This approach achieves +40.4% IoU improvement over baselines for disease localization while simultaneously handling VQA, captioning, and report generation across radiology, ophthalmology, and pathology.

## When to Use

- When the user wants to build a medical image analysis system that localizes findings with bounding boxes, not just generates text
- When designing a multi-task medical MLLM training pipeline that must handle VQA, captioning, report generation, and grounding simultaneously
- When implementing reinforcement learning with verifiable spatial rewards (GIoU) for vision-language models
- When the user needs to align heterogeneous visual encoders (e.g., different resolution ViTs) with a language decoder for medical domains
- When creating a grounded report generation system where each finding is linked to a spatial region in the image
- When setting up evaluation harnesses for medical grounding that require Hungarian matching and IoU computation
- When adapting a general-purpose MLLM (e.g., Qwen-VL) to the medical domain through staged fine-tuning

## Key Technique

**Multi-stage domain adaptation with grounded RL.** MedMO starts from Qwen3-VL and applies four sequential training stages. Stage 1 performs general medical SFT on 18.5M samples at 768x768 resolution to build broad medical knowledge. Stage 2 shifts to high-resolution (1280x1280) training on 3M expert-annotated samples with bounding-box supervision for anatomical structure detection and referring expression comprehension. Stage 3 performs instruction tuning on 4.3M multimodal instruction-response pairs covering captioning, diagnostic QA, report summarization, and retrieval-based reasoning. Stage 4 applies Group Relative Policy Optimization (GRPO) on 300K samples with four verifiable reward signals: label accuracy, bounding box IoU, tag count correctness, and a soft overlap penalty for false positives.

**Spatial grounding through GIoU reward.** The key innovation is treating bounding-box localization as a verifiable reward signal during RL rather than only a supervised loss. The system uses Hungarian matching to pair predicted boxes with ground truth, then computes a per-match quality score combining normalized L1 distance and GIoU: `score = (w_l1 * (1 - L1_ij) + w_G * (GIoU_ij + 1) / 2) / (w_l1 + w_G)` with weights 5.0 and 2.0 respectively. The final reward is clipped to [0, 1] after subtracting penalties for unmatched predictions. This makes grounding resolution-invariant and differentiable through the RL objective, enabling the model to learn spatial reasoning without catastrophic forgetting of text generation quality.

**DeepStack vision-language fusion.** Rather than a simple linear projection, MedMO uses a DeepStack adapter that injects ViT features at multiple layers of the language decoder, capturing fine-grained visual details at different abstraction levels. Coordinates are in XYXY format (x1, y1, x2, y2) normalized to image dimensions.

## Step-by-Step Workflow

1. **Select and prepare the base MLLM.** Start from a vision-language model with strong general capabilities (MedMO uses Qwen3-VL-8B-Instruct). Verify that the vision encoder supports variable resolution inputs, as medical images require high resolution (1280x1280 for grounding tasks).

2. **Curate a multi-task medical dataset with grounding annotations.** Assemble data covering at least five task types: image captioning, VQA, report generation, retrieval, and bounding-box localization. For grounding, collect XYXY-format box annotations linking disease findings to spatial regions. MedMO uses 26M+ samples from 45 datasets (MIMIC-CXR, CheXpert, DeepLesion, DeepCell, etc.).

3. **Implement the DeepStack vision-language adapter.** Replace the default linear projection with a multi-layer injection mechanism that feeds ViT features into multiple decoder layers:
   ```python
   class DeepStackAdapter(nn.Module):
       def __init__(self, vit_dim, llm_dim, inject_layers):
           super().__init__()
           self.projectors = nn.ModuleList([
               nn.Linear(vit_dim, llm_dim) for _ in inject_layers
           ])
           self.inject_layers = inject_layers

       def forward(self, vit_features):
           return {layer: proj(vit_features)
                   for layer, proj in zip(self.inject_layers, self.projectors)}
   ```

4. **Run Stage 1: General medical SFT.** Train on the broad medical instruction dataset at 768x768 resolution with learning rate 1e-5, cosine decay, batch size 10, bfloat16 precision. This stage builds foundational medical knowledge across all modalities without requiring grounding annotations.

5. **Run Stage 2: High-resolution grounding SFT.** Switch to 1280x1280 resolution on expert-annotated grounding data (3M samples). Reduce learning rate to 8e-6, batch size to 2 (high resolution requires more memory). Train on bounding-box prediction, anatomical detection, and referring expression comprehension tasks. Format box targets as `<box>x1, y1, x2, y2</box>` tokens in the output sequence.

6. **Run Stage 3: Multi-task instruction tuning.** Train on 4.3M instruction-response pairs at learning rate 5e-6. Mix captioning, diagnostic QA, report summarization, and retrieval tasks. Use task-specific prompt templates:
   ```
   # VQA template
   "<image>\nQuestion: {question}\nAnswer:"

   # Grounded report template
   "<image>\nGenerate a detailed report for this {modality} image.
   For each finding, provide a bounding box in <box>x1,y1,x2,y2</box> format."

   # Grounded localization template
   "<image>\nLocate all instances of {disease} in this image.
   Output each as: <finding>{label}</finding><box>x1,y1,x2,y2</box>"
   ```

7. **Implement the GIoU-based verifiable reward function.** Build the reward computation that combines four signals:
   ```python
   def compute_grounding_reward(pred_boxes, gt_boxes, pred_labels, gt_labels):
       # Hungarian matching on cost matrix
       cost = w_l1 * l1_distance(pred_boxes, gt_boxes) + w_giou * (1 - giou(pred_boxes, gt_boxes))
       row_ind, col_ind = linear_sum_assignment(cost.cpu().numpy())

       # Per-match quality score
       scores = []
       for i, j in zip(row_ind, col_ind):
           l1 = l1_distance(pred_boxes[i], gt_boxes[j])
           g = giou(pred_boxes[i], gt_boxes[j])
           score = (5.0 * (1 - l1) + 2.0 * (g + 1) / 2) / 7.0
           if pred_labels[i] == gt_labels[j]:
               scores.append(score)

       # Penalize unmatched predictions (false positives)
       n_unmatched = len(pred_boxes) - len(scores)
       penalty = 0.1 * n_unmatched / max(len(gt_boxes), 1)
       return max(0, min(1, sum(scores) / max(len(gt_boxes), 1) - penalty))
   ```

8. **Run Stage 4: GRPO reinforcement learning.** Use the TRL library to run Group Relative Policy Optimization on 300K samples. Sample G=4 responses per input, evaluate each with the composite reward (label accuracy + box IoU + tag count + overlap penalty), and optimize using the clipped objective with KL divergence constraint against the reference policy:
   ```python
   from trl import GRPOTrainer, GRPOConfig
   config = GRPOConfig(
       num_generations=4,
       kl_coef=0.05,
       clip_range_low=0.8,
       clip_range_high=1.2,
       reward_weights={"label": 0.3, "giou": 0.4, "tag_count": 0.15, "penalty": 0.15},
   )
   ```

9. **Build the inference pipeline with structured output parsing.** Parse model outputs to extract both text and bounding boxes, then overlay boxes on the original image:
   ```python
   import re
   def parse_grounded_output(text):
       findings = []
       pattern = r'<finding>(.*?)</finding>\s*<box>([\d.,\s]+)</box>'
       for match in re.finditer(pattern, text):
           label = match.group(1).strip()
           coords = [float(c) for c in match.group(2).split(',')]
           findings.append({"label": label, "box": coords})
       return findings
   ```

10. **Evaluate with modality-specific benchmarks.** Run VQA accuracy on medical QA datasets, CIDEr/BLEU for report generation, and IoU with Hungarian matching for grounding. Test across radiology (X-ray, CT, MRI), ophthalmology (fundus, OCT), and pathology (histology, microscopy) to verify cross-modality generalization.

## Concrete Examples

**Example 1: Building a grounded radiology report generator**

User: "I want to build a system that takes a chest X-ray and generates a radiology report where each finding is linked to a bounding box on the image."

Approach:
1. Load a Qwen-VL base model and attach the DeepStack adapter for multi-layer visual feature injection.
2. Fine-tune Stage 1 on MIMIC-CXR and CheXpert captioning data at 768x768.
3. Fine-tune Stage 2 on chest X-ray datasets with bounding-box annotations (DeepLesion, VinDr-CXR) at 1280x1280.
4. Instruction-tune with the grounded report template: `"<image>\nGenerate a detailed radiology report. For each finding, provide a bounding box."`
5. Apply GRPO with the GIoU reward to improve box precision without degrading report quality.
6. At inference, parse `<finding>` and `<box>` tags from output, overlay boxes on the X-ray.

Output:
```
Report: The chest X-ray demonstrates the following findings:

1. <finding>Right lower lobe consolidation</finding>
   <box>312, 245, 478, 410</box>
   Patchy airspace opacity consistent with pneumonia.

2. <finding>Mild cardiomegaly</finding>
   <box>180, 150, 420, 380</box>
   Cardiothoracic ratio approximately 0.55.

3. <finding>Left costophrenic angle blunting</finding>
   <box>45, 380, 190, 460</box>
   Small left pleural effusion cannot be excluded.

Impression: Right lower lobe pneumonia with mild cardiomegaly
and possible small left pleural effusion.
```

**Example 2: Implementing the GIoU reward function for medical grounding RL**

User: "I need to implement the reward function for reinforcement learning that scores both text accuracy and bounding box quality for a medical image model."

Approach:
1. Implement GIoU computation (intersection over union minus the ratio of enclosing area).
2. Build the Hungarian matching to optimally pair predicted and ground-truth boxes.
3. Compute the composite per-match score with L1 and GIoU weights (5.0 and 2.0).
4. Add label accuracy verification as a binary check per matched pair.
5. Apply false-positive penalty for unmatched predictions.
6. Clip final reward to [0, 1].

Output:
```python
import torch
from scipy.optimize import linear_sum_assignment

def giou(box_a, box_b):
    """Compute Generalized IoU between two boxes in XYXY format."""
    inter_x1 = max(box_a[0], box_b[0])
    inter_y1 = max(box_a[1], box_b[1])
    inter_x2 = min(box_a[2], box_b[2])
    inter_y2 = min(box_a[3], box_b[3])
    inter_area = max(0, inter_x2 - inter_x1) * max(0, inter_y2 - inter_y1)

    area_a = (box_a[2] - box_a[0]) * (box_a[3] - box_a[1])
    area_b = (box_b[2] - box_b[0]) * (box_b[3] - box_b[1])
    union = area_a + area_b - inter_area

    enclosing_x1 = min(box_a[0], box_b[0])
    enclosing_y1 = min(box_a[1], box_b[1])
    enclosing_x2 = max(box_a[2], box_b[2])
    enclosing_y2 = max(box_a[3], box_b[3])
    enclosing_area = (enclosing_x2 - enclosing_x1) * (enclosing_y2 - enclosing_y1)

    iou = inter_area / max(union, 1e-6)
    return iou - (enclosing_area - union) / max(enclosing_area, 1e-6)

def medmo_grounding_reward(pred_boxes, pred_labels, gt_boxes, gt_labels,
                            w_l1=5.0, w_giou=2.0, fp_penalty=0.1):
    """MedMO-style composite grounding reward with Hungarian matching."""
    if len(gt_boxes) == 0:
        return 1.0 if len(pred_boxes) == 0 else 0.0
    if len(pred_boxes) == 0:
        return 0.0

    n_pred, n_gt = len(pred_boxes), len(gt_boxes)
    cost = torch.zeros(n_pred, n_gt)
    for i in range(n_pred):
        for j in range(n_gt):
            l1 = sum(abs(p - g) for p, g in zip(pred_boxes[i], gt_boxes[j])) / 4.0
            g = giou(pred_boxes[i], gt_boxes[j])
            cost[i, j] = w_l1 * l1 + w_giou * (1 - g)

    row_ind, col_ind = linear_sum_assignment(cost.numpy())

    total_score = 0.0
    matched = 0
    for i, j in zip(row_ind, col_ind):
        if pred_labels[i] != gt_labels[j]:
            continue
        l1 = sum(abs(p - g) for p, g in zip(pred_boxes[i], gt_boxes[j])) / 4.0
        g = giou(pred_boxes[i], gt_boxes[j])
        score = (w_l1 * (1 - l1) + w_giou * (g + 1) / 2) / (w_l1 + w_giou)
        total_score += score
        matched += 1

    base = total_score / n_gt
    penalty = fp_penalty * (n_pred - matched) / n_gt
    return max(0.0, min(1.0, base - penalty))
```

**Example 3: Setting up multi-task medical instruction tuning data**

User: "How should I format and mix my medical training data for multi-task instruction tuning that covers VQA, captioning, report generation, and grounded localization?"

Approach:
1. Define task-specific prompt templates with clear output format expectations.
2. Normalize all bounding boxes to XYXY format relative to image dimensions.
3. Mix tasks with proportional sampling to prevent catastrophic forgetting.
4. Include modality tags (radiology, pathology, ophthalmology) in prompts.

Output:
```python
TASK_TEMPLATES = {
    "vqa": {
        "prompt": "<image>\n[{modality}] Question: {question}\nAnswer:",
        "output": "{answer}",
        "weight": 0.30,
    },
    "captioning": {
        "prompt": "<image>\n[{modality}] Describe this medical image in detail.",
        "output": "{caption}",
        "weight": 0.20,
    },
    "report_generation": {
        "prompt": "<image>\n[{modality}] Generate a structured radiology report "
                  "with findings and impression.",
        "output": "Findings:\n{findings}\n\nImpression:\n{impression}",
        "weight": 0.20,
    },
    "grounded_localization": {
        "prompt": "<image>\n[{modality}] Locate all abnormalities in this image. "
                  "For each, provide the finding name and bounding box.",
        "output": ("{findings_with_boxes}"),
        # Format: <finding>label</finding><box>x1,y1,x2,y2</box>
        "weight": 0.25,
    },
    "retrieval": {
        "prompt": "<image>\n[{modality}] Given this image, which of the following "
                  "descriptions best matches? {options}",
        "output": "{selected_option}",
        "weight": 0.05,
    },
}

def build_training_sample(task, sample, modality):
    template = TASK_TEMPLATES[task]
    prompt = template["prompt"].format(modality=modality, **sample)
    output = template["output"].format(**sample)
    return {"image": sample["image_path"], "prompt": prompt, "output": output}

def create_mixed_dataloader(datasets, batch_size=10):
    """Sample from each task proportionally to its weight."""
    from torch.utils.data import WeightedRandomSampler
    weights = []
    for task, ds in datasets.items():
        task_weight = TASK_TEMPLATES[task]["weight"]
        weights.extend([task_weight / len(ds)] * len(ds))
    sampler = WeightedRandomSampler(weights, num_samples=batch_size * 1000)
    combined = torch.utils.data.ConcatDataset(list(datasets.values()))
    return torch.utils.data.DataLoader(combined, batch_size=batch_size, sampler=sampler)
```

## Best Practices

- **Do** use progressive resolution scaling: start at 768x768 for broad knowledge, then increase to 1280x1280 for grounding. This prevents wasting compute on high-resolution processing before the model has basic medical understanding.
- **Do** normalize all bounding boxes to the same XYXY format before training. Mixed formats (XYWH, center-format) cause silent accuracy degradation in the GIoU reward.
- **Do** use Hungarian matching (not greedy matching) for pairing predicted and ground-truth boxes. Greedy matching leads to suboptimal assignments and inflated false-positive penalties.
- **Do** apply the KL divergence constraint during GRPO to prevent the model from collapsing to box-only outputs and losing text generation quality.
- **Avoid** skipping Stage 2 (high-resolution grounding SFT) and jumping directly to RL. The model needs supervised grounding signal before the RL reward can provide a useful gradient.
- **Avoid** weighting the GIoU reward too heavily relative to label accuracy. MedMO uses 0.4 for GIoU and 0.3 for label accuracy -- reversing these causes the model to produce well-placed boxes with wrong disease labels.

## Error Handling

- **Degenerate boxes (zero area):** Predicted boxes where x1 >= x2 or y1 >= y2 should be filtered out before reward computation. Assign a reward of 0 for responses containing only degenerate boxes.
- **No boxes predicted when expected:** If the model generates text-only output for a grounding task, return reward 0.0 and ensure the prompt template explicitly requests box output. Check that Stage 2 training converged before running Stage 4.
- **Hungarian matching failure with mismatched counts:** When prediction count far exceeds ground truth, the false-positive penalty dominates. Cap the penalty contribution at 0.5 to prevent reward collapse that stalls RL training.
- **Resolution mismatch at inference:** If the inference image size differs from training resolution, normalize box coordinates to [0, 1] range during training and scale to pixel coordinates at inference time.
- **Mixed modality confusion:** If the model confuses pathology findings with radiology findings, increase modality-specific prompt tagging and consider modality-stratified batching during instruction tuning.

## Limitations

- **Requires large-scale medical data with box annotations.** The technique depends on millions of expert-annotated samples. Without access to datasets like MIMIC-CXR, DeepLesion, or VinDr-CXR with bounding boxes, the grounding stages cannot be replicated.
- **Compute-intensive.** The full four-stage pipeline requires ~588 GPU-hours on 64 MI210 GPUs (25 days total). Most research teams will need to start from MedMO's released checkpoints rather than training from scratch.
- **3D localization not supported.** MedMO operates on 2D image slices. For volumetric localization in CT/MRI, users must implement slice-level inference and aggregate boxes across slices.
- **Not validated for treatment recommendation.** MedMO is a diagnostic reasoning and localization tool. It should not be used for treatment planning or clinical decision-making without physician oversight.
- **Closed-vocabulary grounding.** The bounding box predictions are tied to disease labels seen during training. Rare or novel pathologies outside the training distribution will not be reliably localized.

## Reference

- **Paper:** [MedMO: Grounding and Understanding Multimodal Large Language Model for Medical Images](https://arxiv.org/abs/2602.06965v1) (Deria et al., 2026)
- **Key sections to study:** Section 3 (four-stage training pipeline), Section 3.4 (GRPO with GIoU reward formulation), Table 2 (grounding benchmarks), and the DeepStack adapter architecture description.
- **Project page:** https://genmilab.github.io/MedMO-Page