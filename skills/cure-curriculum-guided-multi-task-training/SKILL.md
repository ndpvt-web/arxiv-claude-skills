---
name: "cure-curriculum-guided-multi-task-training"
description: "Implement error-aware curriculum learning for multi-task training pipelines, especially medical/vision-language models. Dynamically reweight training samples by difficulty to improve grounding accuracy and reduce hallucinations. Use when: 'set up curriculum learning for my training pipeline', 'reweight samples by difficulty', 'reduce hallucinations in my vision-language model', 'implement error-aware sampling for multi-task fine-tuning', 'train a grounded report generation model', 'build an anatomy-grounded radiology model with CURE'."
---

# CURE: Curriculum-Guided Multi-Task Training for Reliable Grounded Generation

This skill teaches Claude to implement the CURE framework — an error-aware curriculum learning strategy that dynamically adjusts training sample weights across multiple tasks based on per-category model error. Originally designed for radiology report generation with visual grounding, the core technique generalizes to any multi-task training scenario where some categories or datasets are harder than others and you want data-efficient improvement without collecting new data. CURE's key insight: measure per-sample or per-category error after a warm-up phase, then redistribute sampling probability proportionally to error so the model spends more time on what it gets wrong.

## When to Use

- When the user is building a multi-task training pipeline and wants to prioritize harder examples automatically
- When fine-tuning a vision-language model (e.g., MedGemma, LLaVA, PaliGemma) and needs to reduce hallucinations
- When training on heterogeneous datasets with uneven difficulty across categories or anatomy regions
- When implementing curriculum learning with dynamic reweighting in PyTorch or HuggingFace Trainer
- When the user wants to reproduce or adapt the CURE pipeline for grounded report generation on chest X-rays
- When building a radiology AI system that must align textual findings with bounding boxes on medical images

## Key Technique

CURE reformulates three related tasks — phrase grounding (PG), grounded report generation (GRG), and anatomy-grounded report generation (AGRG) — into a unified `(image, instruction, response)` triplet format. This lets a single instruction-tuned model handle bounding box prediction, report text generation, and anatomy localization through different prompt templates rather than separate model heads. The base model is fine-tuned with LoRA (rank 16, 4-bit NF4 quantization) so the entire pipeline runs on a single GPU.

The curriculum strategy operates in two phases. First, a **warm-up phase** (3000 steps) trains with uniform sampling across all datasets and categories to establish a baseline. Then the model is evaluated per-dataset and per-category using an aggregate score `s_i = alpha * IoU_i + (1 - alpha) * CXRFEScore_i` (alpha=0.8 to prioritize spatial accuracy). The error `e_i = 1 - s_i` is normalized into sampling probabilities `p_i = e_i / sum(e_j)`. This reweighting happens at two levels: **inter-dataset** (which dataset gets more samples) and **intra-dataset** (which categories within a dataset get more samples). A second training phase (3000 steps) then trains with these error-proportional weights, forcing the model to focus on its weakest areas.

What makes this approach powerful is its simplicity and data efficiency. There is no new data, no architectural modification, and no complex loss weighting — just a principled resampling strategy based on measured error. The result is +0.37 IoU improvement in grounding accuracy, +0.188 CXRFEScore in report quality, and 18.6% fewer hallucinations compared to the baseline, all from the same training data.

## Step-by-Step Workflow

1. **Unify task formats into instruction triplets.** Convert each task into `(image, instruction, response)` tuples. For phrase grounding: instruction is `"Ground the phrase: {phrase}"`, response is `"{phrase} [cx,cy,w,h]"`. For report generation: instruction is `"Generate a grounded report"`, response is the report text with inline bounding boxes. For anatomy tasks: instruction is `"Locate the {region}"`, `"Describe the {region}"`, or `"Locate and describe the {region}"`.

2. **Prepare dataset mixtures with metadata.** Tag each training sample with its source dataset and category (e.g., anatomical region, finding type). Store these tags in a metadata column so the sampler can group and reweight by category. Build a `ConcatDataset` or equivalent that merges all task datasets.

3. **Configure LoRA fine-tuning on the base model.** Apply LoRA adapters (rank 16) to the base instruction-tuned model. Use 4-bit NF4 quantization with BF16 compute dtype. Set learning rate to 2e-4 with linear scheduler and 3% warmup ratio. Use gradient accumulation (5 steps x 5 per-device = 25 effective batch size).

4. **Run warm-up training with uniform sampling (Phase 1).** Train for 3000 steps with uniform sampling across all datasets and categories. This establishes a performance baseline and ensures the model sees all data types before reweighting begins.

5. **Evaluate per-dataset and per-category performance.** After warm-up, run inference on validation subsets. Compute IoU for spatial tasks and CXRFEScore (or your domain-specific semantic metric) for text tasks. Calculate the aggregate score per category: `s_i = 0.8 * IoU_i + 0.2 * SemanticScore_i`.

6. **Compute error-proportional sampling weights.** For each dataset/category, compute error `e_i = 1 - s_i`. Normalize to probabilities: `p_i = e_i / sum(e_j)`. These become the new sampling weights for the next training phase.

7. **Build a weighted sampler from the error distribution.** Implement a `WeightedRandomSampler` (or custom batch sampler) that assigns each sample the probability of its category. This operates at both inter-dataset and intra-dataset granularity.

8. **Run curriculum training with error-weighted sampling (Phase 2).** Train for another 3000 steps using the reweighted sampler. The model now trains disproportionately on categories where it performed worst, improving its weakest areas without neglecting strong ones.

9. **Apply data augmentation to training images.** Use CLAHE contrast enhancement, random spatial transforms (affine with +/-10% scaling, +/-15% rotation), and resize to 448x448. This is especially important for medical imaging where contrast varies.

10. **Evaluate final model on held-out and out-of-domain data.** Measure IoU, CXRFEScore, CheXbert F1, RadGraph F1, and hallucination rate (via NLI against ground truth). Test zero-shot on an out-of-domain dataset (e.g., VinDr-CXR) to verify generalization.

## Concrete Examples

**Example 1: Implementing the Error-Aware Sampler in PyTorch**

User: "I have three datasets for my multi-task medical VLM training. How do I implement CURE's error-aware curriculum sampling?"

Approach:
1. Define a category-aware dataset wrapper that exposes category IDs
2. After warm-up training, evaluate per-category metrics
3. Build a WeightedRandomSampler from the error distribution

Output:
```python
import torch
from torch.utils.data import WeightedRandomSampler, DataLoader

def compute_curriculum_weights(eval_results: dict, alpha: float = 0.8) -> dict:
    """
    Compute CURE-style error-proportional sampling weights.

    Args:
        eval_results: {category_id: {"iou": float, "semantic_score": float}}
        alpha: weight for spatial vs semantic score (0.8 in CURE)

    Returns:
        {category_id: sampling_probability}
    """
    errors = {}
    for cat_id, metrics in eval_results.items():
        aggregate_score = alpha * metrics["iou"] + (1 - alpha) * metrics["semantic_score"]
        errors[cat_id] = 1.0 - aggregate_score

    total_error = sum(errors.values())
    if total_error == 0:
        # Uniform fallback if model is perfect everywhere
        n = len(errors)
        return {cat_id: 1.0 / n for cat_id in errors}

    return {cat_id: e / total_error for cat_id, e in errors.items()}


def build_curriculum_sampler(dataset, category_weights: dict) -> WeightedRandomSampler:
    """
    Build a weighted sampler where each sample's weight comes from its category error.

    Args:
        dataset: must have a `get_category(idx)` method returning category_id
        category_weights: output of compute_curriculum_weights
    """
    sample_weights = []
    for idx in range(len(dataset)):
        cat_id = dataset.get_category(idx)
        sample_weights.append(category_weights.get(cat_id, 1.0 / len(category_weights)))

    return WeightedRandomSampler(
        weights=sample_weights,
        num_samples=len(dataset),
        replacement=True,
    )


# Usage in training loop:
# Phase 1: warm-up with uniform sampling
warmup_loader = DataLoader(combined_dataset, batch_size=5, shuffle=True)
train_for_steps(model, warmup_loader, num_steps=3000)

# Evaluate per-category
eval_results = evaluate_per_category(model, val_dataset)
# e.g., {"right_lung": {"iou": 0.45, "semantic_score": 0.72}, ...}

# Phase 2: curriculum-weighted sampling
weights = compute_curriculum_weights(eval_results, alpha=0.8)
curriculum_sampler = build_curriculum_sampler(combined_dataset, weights)
curriculum_loader = DataLoader(combined_dataset, batch_size=5, sampler=curriculum_sampler)
train_for_steps(model, curriculum_loader, num_steps=3000)
```

**Example 2: Formatting Multi-Task Instruction Triplets**

User: "How do I format my chest X-ray grounding and report data into CURE's unified instruction format?"

Approach:
1. Define prompt templates for each of the three task types
2. Normalize bounding box coordinates to [0, 1] range
3. Produce (image_path, instruction, response) triplets

Output:
```python
TASK_TEMPLATES = {
    "phrase_grounding": {
        "instruction": "Ground the phrase: {phrase}",
        "response": "{phrase} [{cx:.3f},{cy:.3f},{w:.3f},{h:.3f}]",
    },
    "grounded_report": {
        "instruction": "Generate a grounded report for this chest X-ray.",
        "response": "{report_with_inline_boxes}",
    },
    "locate_anatomy": {
        "instruction": "Locate the {anatomy}",
        "response": "{anatomy} [{cx:.3f},{cy:.3f},{w:.3f},{h:.3f}]",
    },
    "describe_anatomy": {
        "instruction": "Describe the {anatomy}",
        "response": "{description}",
    },
    "locate_and_describe": {
        "instruction": "Locate and describe the {anatomy}",
        "response": "{anatomy} [{cx:.3f},{cy:.3f},{w:.3f},{h:.3f}] {description}",
    },
}

def normalize_bbox(bbox, img_w, img_h):
    """Convert [x1, y1, x2, y2] pixel coords to normalized [cx, cy, w, h]."""
    x1, y1, x2, y2 = bbox
    cx = ((x1 + x2) / 2) / img_w
    cy = ((y1 + y2) / 2) / img_h
    w = (x2 - x1) / img_w
    h = (y2 - y1) / img_h
    return [cx, cy, w, h]

def make_triplet(task_type, image_path, **kwargs):
    """Create a unified (image, instruction, response) triplet."""
    template = TASK_TEMPLATES[task_type]
    instruction = template["instruction"].format(**kwargs)
    response = template["response"].format(**kwargs)
    return {"image": image_path, "instruction": instruction, "response": response}
```

**Example 3: Adapting CURE's Curriculum to a Non-Medical Multi-Task Setting**

User: "I'm training a model on three tasks — object detection, captioning, and VQA. Can I use CURE's curriculum approach?"

Approach:
1. Map each task to a measurable metric (mAP for detection, CIDEr for captioning, accuracy for VQA)
2. Normalize scores to [0, 1] range
3. Apply the same error-proportional reweighting

Output:
```python
# Define per-task metrics (all normalized to [0, 1])
eval_results_by_task = {
    "object_detection": {"score": 0.62},  # mAP
    "captioning":       {"score": 0.74},  # CIDEr / max_possible
    "vqa":              {"score": 0.81},  # accuracy
}

# Compute inter-task curriculum weights
errors = {task: 1.0 - m["score"] for task, m in eval_results_by_task.items()}
total = sum(errors.values())
task_weights = {task: e / total for task, e in errors.items()}
# Result: detection gets ~52%, captioning ~36%, VQA ~12% of samples
# The model focuses most on its weakest task (detection) automatically

# For intra-task reweighting, compute per-category errors within each task
# e.g., within object_detection, "small objects" has IoU=0.3 vs "large objects" IoU=0.8
# Apply the same formula at the category level
```

## Best Practices

- **Do:** Always start with a uniform-sampling warm-up phase before reweighting. The model needs a baseline before you can meaningfully measure per-category error.
- **Do:** Operate curriculum reweighting at two levels — inter-dataset (which task gets more samples) and intra-dataset (which categories within a task). This is what gives CURE its fine-grained improvement.
- **Do:** Use a high alpha (0.8) to prioritize spatial/structural metrics over semantic ones when grounding accuracy is critical. Adjust alpha based on which metric matters more for your use case.
- **Do:** Include both normal and abnormal examples per category. CURE's anatomy-grounded task exposes the model to normal findings, which reduces false-positive hallucinations.
- **Avoid:** Skipping the warm-up phase. Reweighting from step zero leads to unstable training because initial errors are noisy and uninformative.
- **Avoid:** Running too many curriculum cycles. CURE uses just one reweighting step (warm-up then curriculum). Repeated reweighting risks overfitting to the hardest samples and forgetting easier but important patterns.
- **Avoid:** Using this approach when you have very few categories (< 3). The reweighting becomes trivial and the overhead is not justified.

## Error Handling

- **Degenerate weights:** If one category has error near 1.0 and all others near 0.0, the sampler will almost exclusively draw from that category. Cap maximum weight at 5-10x the uniform baseline to prevent catastrophic forgetting of well-learned categories.
- **Empty categories in evaluation:** If a category has zero validation samples, assign it the mean error of other categories rather than dropping it.
- **Metric incompatibility across tasks:** When combining IoU (spatial) and text-quality scores (semantic), ensure both are in [0, 1]. If your semantic metric is unbounded (e.g., BLEU unnormalized), clip or rescale before computing the aggregate score.
- **Checkpoint before curriculum phase:** Save the model after warm-up. If curriculum training degrades performance (rare but possible with aggressive weighting), you can roll back and adjust alpha or cap weights.

## Limitations

- The technique assumes you can meaningfully evaluate per-category performance on a validation set. If your categories are too fine-grained or validation data is sparse, error estimates will be noisy.
- CURE was validated on chest X-ray datasets (MS-CXR, PadChest-GR, Chest ImaGenome). While the curriculum principle generalizes, the specific prompt templates and metrics (IoU, CXRFEScore) are domain-specific.
- A single reweighting step may be insufficient for extremely imbalanced difficulty distributions. The paper does not explore iterative multi-round curriculum schedules.
- The approach does not address label noise — if hard samples are hard because they are mislabeled, upweighting them will degrade the model.
- LoRA fine-tuning at rank 16 with 4-bit quantization is efficient but may limit the model's capacity ceiling compared to full fine-tuning on larger compute budgets.

## Reference

**Paper:** [CURE: Curriculum-guided Multi-task Training for Reliable Anatomy Grounded Report Generation](https://arxiv.org/abs/2601.15408v1) (Messina et al., 2026). Look for Section 3 (Method) for the curriculum formulation, Table 2 for the aggregate score formula, and Section 4 for ablation studies showing the contribution of each component. Code: [github.com/PabloMessina/CURE](https://github.com/PabloMessina/CURE). Model: [huggingface.co/pamessina/medgemma-4b-it-cure](https://huggingface.co/pamessina/medgemma-4b-it-cure).