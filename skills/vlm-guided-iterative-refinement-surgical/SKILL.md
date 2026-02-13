---
name: "vlm-guided-iterative-refinement-surgical"
description: "Build iterative VLM-guided refinement pipelines for image segmentation tasks, especially surgical/medical imagery. Uses a foundation segmentation model for initial prediction, a VLM for quality assessment and instrument detection, and an agentic loop that selects refinement strategies based on coverage metrics. Trigger phrases: 'iterative segmentation refinement', 'VLM-guided segmentation', 'surgical image segmentation pipeline', 'self-refining segmentation agent', 'language-guided segmentation with feedback', 'build a segmentation refinement loop'"
---

# VLM-Guided Iterative Refinement for Image Segmentation

This skill teaches Claude to build **agentic iterative refinement pipelines** for image segmentation, following the IR-SIS architecture from Lou et al. (2026). The core idea: instead of treating segmentation as a one-shot prediction, use a Vision-Language Model to assess the quality of an initial segmentation, then adaptively select a refinement strategy (morphological post-processing vs. re-segmentation) and loop until quality thresholds are met or a clinician provides feedback. This pattern generalizes beyond surgery to any domain where segmentation quality matters and human-in-the-loop correction is valuable.

## When to Use

- When the user wants to build a **segmentation pipeline with automatic quality assessment and iterative correction**, not just a single forward pass
- When the user asks to combine a **foundation segmentation model (SAM, SAM2, SAM3)** with a **VLM (Qwen-VL, GPT-4o, etc.)** for detection or evaluation
- When building a **language-guided segmentation** system that accepts natural language descriptions of target objects instead of fixed class labels
- When implementing a **human-in-the-loop refinement workflow** where clinicians or annotators can provide text/box corrections that override automatic assessment
- When the user needs to handle **multi-instrument or multi-object segmentation** where some objects are well-segmented and others need re-processing
- When designing **agentic workflows** that decide between multiple refinement strategies based on computed quality metrics

## Key Technique

**The IR-SIS architecture** decomposes segmentation into four stages: (1) language-based initial segmentation via a fine-tuned SAM3, (2) VLM-based instrument detection producing bounding boxes, (3) quality evaluation using coverage metrics, and (4) an agentic refinement loop. The critical insight is that a VLM can serve as both a **detector** (finding what should be segmented) and a **quality judge** (assessing whether the segmentation actually covers those detections). This dual role eliminates the need for ground-truth labels at inference time.

**Quality assessment** uses two complementary metrics. **Mask Coverage (C)** measures what fraction of all VLM-detected bounding boxes is covered by the current segmentation mask: `C = |M ∩ ∪bᵢ| / |∪bᵢ|`. **Mask-Box Overlap (O)** measures alignment between the mask and a specific target bounding box: `O = |M ∩ b_target| / |b_target|`. A binary quality indicator `S = H(C > τ_c) AND H(O > τ_o)` determines whether refinement is needed. When `S=1`, the system trusts the initial prediction and applies only morphological cleanup. When `S=0`, it re-segments low-quality regions using box prompts derived from the VLM detections, preserving high-quality regions.

**Multi-granularity language annotations** are another key contribution. Instead of single-label training, each object gets three annotation levels: general ("surgical instrument"), category ("forceps"), and specific ("bipolar forceps"). This 3x data expansion teaches the model to respond to flexible natural language queries at any specificity level, which is essential for real-world use where users describe targets imprecisely.

## Step-by-Step Workflow

1. **Set up the segmentation backbone.** Fine-tune SAM3 (or SAM2) with a hierarchical learning rate: low rate for the vision encoder (e.g., 2.5e-5 with 0.98 layer decay), higher rate for the decoder (e.g., 8e-5), and freeze the text encoder entirely. Train at high resolution (1008x1008) with domain-specific augmentation (color jitter, gamma correction, specular highlights for surgical data).

2. **Construct multi-granularity language annotations.** For each object instance in your dataset, create three text prompts at increasing specificity. For surgical data: Level 0 = "surgical instrument", Level 1 = "grasper", Level 2 = "large needle driver on the left". Each instance produces three training samples, tripling dataset size.

3. **Design the composite loss function.** Combine four losses with tuned weights: focal loss for mask prediction (weight=5, handles class imbalance), dice loss (weight=1, improves small object segmentation), cross-entropy for query-object matching (weight=2), and a presence prediction loss (weight=2, predicts whether the queried object exists in the image at all).

4. **Implement VLM-based detection.** At inference, send the input image to a VLM (e.g., Qwen-2.5-VL-32B) with a detection prompt asking it to identify all target objects and output bounding boxes. Parse the VLM response into a set of bounding boxes `B = {b_1, ..., b_n}`.

5. **Run initial segmentation.** Pass the image and the user's natural language query to the fine-tuned SAM model. Obtain the initial mask `M_0`.

6. **Compute quality metrics.** Calculate Mask Coverage `C` (intersection of mask with union of all detected boxes, divided by union area) and Mask-Box Overlap `O` (intersection of mask with target box, divided by target box area). Compare against thresholds `τ_c` and `τ_o` (tune on validation data).

7. **Select refinement strategy.** If `S=1` (quality passes): apply morphological post-processing only — opening to remove small noise regions, closing to fill holes. If `S=0` (quality fails): identify which bounding boxes have low overlap, re-segment those regions using box prompts, and merge with the high-quality portions of the existing mask.

8. **Iterate the refinement loop.** After each re-segmentation, recompute quality metrics. Continue until `S=1` or a maximum iteration count is reached (typically 2-3 iterations suffice). Each iteration only re-processes failing regions, preserving already-good segments.

9. **Support human-in-the-loop feedback.** Accept three types of corrections: bounding box prompts (user draws a box around a missed object), language descriptions ("you missed the grasper on the right"), or reference annotations. Human feedback overrides the VLM assessment and triggers targeted re-segmentation.

10. **Return the final mask with confidence metadata.** Output the refined segmentation mask along with per-region quality scores (the overlap metric O for each detected object), the number of refinement iterations performed, and any VLM detection results, so downstream systems or users can assess reliability.

## Concrete Examples

**Example 1: Building a surgical segmentation pipeline with iterative refinement**

User: "I want to build a pipeline that segments surgical instruments from endoscopic images using natural language queries. It should automatically detect if the segmentation is bad and fix it."

Approach:
1. Fine-tune SAM3 on EndoVis2017/2018 with multi-granularity annotations (general, category, specific)
2. Set up Qwen-2.5-VL as the detection/assessment VLM
3. Implement the two-metric quality check (coverage + overlap)
4. Wire up the refinement loop with strategy selection

```python
import torch
import numpy as np
from sam3 import SAM3Model
from qwen_vl import QwenVL

class IRSISPipeline:
    def __init__(self, sam_checkpoint, vlm_model="Qwen/Qwen2.5-VL-32B-Instruct",
                 coverage_threshold=0.7, overlap_threshold=0.6, max_iters=3):
        self.sam = SAM3Model.load(sam_checkpoint)
        self.vlm = QwenVL(vlm_model)
        self.tau_c = coverage_threshold
        self.tau_o = overlap_threshold
        self.max_iters = max_iters

    def detect_instruments(self, image):
        """Use VLM to detect all instruments and return bounding boxes."""
        prompt = (
            "Identify every surgical instrument visible in this endoscopic image. "
            "For each instrument, output its name and bounding box as [x1,y1,x2,y2] "
            "in pixel coordinates."
        )
        response = self.vlm.query(image, prompt)
        return self._parse_boxes(response)  # returns list of (name, [x1,y1,x2,y2])

    def compute_quality(self, mask, boxes, target_box):
        """Compute coverage C and overlap O metrics."""
        union_box_mask = np.zeros_like(mask)
        for _, box in boxes:
            x1, y1, x2, y2 = box
            union_box_mask[y1:y2, x1:x2] = 1
        C = np.sum(mask & union_box_mask) / max(np.sum(union_box_mask), 1)

        target_mask = np.zeros_like(mask)
        x1, y1, x2, y2 = target_box
        target_mask[y1:y2, x1:x2] = 1
        O = np.sum(mask & target_mask) / max(np.sum(target_mask), 1)

        quality_pass = (C > self.tau_c) and (O > self.tau_o)
        return C, O, quality_pass

    def morphological_cleanup(self, mask, open_k=5, close_k=7):
        """Trust-initial strategy: opening then closing."""
        import cv2
        kernel_open = cv2.getStructuringElement(cv2.MORPH_ELLIPSE, (open_k, open_k))
        kernel_close = cv2.getStructuringElement(cv2.MORPH_ELLIPSE, (close_k, close_k))
        mask = cv2.morphologyEx(mask.astype(np.uint8), cv2.MORPH_OPEN, kernel_open)
        mask = cv2.morphologyEx(mask, cv2.MORPH_CLOSE, kernel_close)
        return mask.astype(bool)

    def segment(self, image, query, clinician_feedback=None):
        """Full iterative refinement pipeline."""
        # Stage 1: Initial segmentation
        mask = self.sam.predict(image, text_prompt=query)

        # Stage 2: VLM detection
        detections = self.detect_instruments(image)
        if not detections:
            return self.morphological_cleanup(mask), {"iters": 0, "quality": "no_detections"}

        target_box = self._match_query_to_box(query, detections)

        # Stage 3-4: Iterative refinement
        for i in range(self.max_iters):
            C, O, quality_pass = self.compute_quality(mask, detections, target_box)

            if clinician_feedback:
                # Human feedback overrides VLM assessment
                mask = self._apply_feedback(image, mask, clinician_feedback)
                clinician_feedback = None
                continue

            if quality_pass:
                return self.morphological_cleanup(mask), {"iters": i, "C": C, "O": O}

            # Multi-instrument strategy: re-segment low-quality regions
            low_quality_boxes = self._find_low_overlap_boxes(mask, detections)
            for name, box in low_quality_boxes:
                region_mask = self.sam.predict(image, box_prompt=box)
                mask = self._merge_masks(mask, region_mask, box)

        return self.morphological_cleanup(mask), {"iters": self.max_iters, "C": C, "O": O}
```

**Example 2: Adding multi-granularity annotations to an existing dataset**

User: "I have a dataset of 1000 annotated surgical images with instance-level labels. How do I create the multi-granularity text annotations for training?"

Approach:
1. Define a 3-level taxonomy mapping specific labels to categories and general terms
2. Generate three text prompts per instance
3. Output an expanded dataset with 3x training samples

```python
TAXONOMY = {
    # specific_label: (category, general)
    "bipolar_forceps": ("forceps", "surgical instrument"),
    "prograsp_forceps": ("forceps", "surgical instrument"),
    "large_needle_driver": ("needle driver", "surgical instrument"),
    "vessel_sealer": ("sealer", "surgical instrument"),
    "grasping_retractor": ("retractor", "surgical instrument"),
    "monopolar_curved_scissors": ("scissors", "surgical instrument"),
    "ultrasound_probe": ("probe", "surgical instrument"),
}

def expand_annotations(dataset):
    """Triple dataset size with multi-granularity text prompts."""
    expanded = []
    for sample in dataset:
        image, mask, label = sample["image"], sample["mask"], sample["label"]
        category, general = TAXONOMY[label]

        # Level 0: General
        expanded.append({"image": image, "mask": mask, "text": general})
        # Level 1: Category
        expanded.append({"image": image, "mask": mask, "text": category})
        # Level 2: Specific
        expanded.append({"image": image, "mask": mask, "text": label.replace("_", " ")})

    return expanded  # 3x original size
```

**Example 3: Adapting the pattern to non-surgical domains (e.g., satellite imagery)**

User: "I like this iterative refinement approach but I'm working on building segmentation, not surgery. Can I adapt it?"

Approach:
1. Replace the domain-specific taxonomy with building types
2. Swap surgical augmentations for satellite-appropriate ones
3. Keep the VLM detection + quality assessment + refinement loop intact

```python
# The architecture is domain-agnostic. Adapt these components:

# 1. Taxonomy for multi-granularity annotations
BUILDING_TAXONOMY = {
    "residential_house": ("residential", "building"),
    "apartment_complex": ("residential", "building"),
    "office_tower": ("commercial", "building"),
    "warehouse": ("industrial", "building"),
}

# 2. Domain-specific augmentations (replace specular highlights with satellite artifacts)
AUGMENTATIONS = {
    "color_jitter": {"p": 0.8},
    "random_rotation": {"p": 0.5, "degrees": 180},  # satellite images have arbitrary orientation
    "cloud_shadow_sim": {"p": 0.3},  # replaces surgical shadow augmentation
    "atmospheric_haze": {"p": 0.2},  # replaces specular highlight simulation
}

# 3. VLM detection prompt (domain-adapted)
DETECTION_PROMPT = (
    "Identify every building visible in this satellite image. "
    "For each building, output its type and bounding box as [x1,y1,x2,y2]."
)

# 4. Quality thresholds (may need different values per domain)
THRESHOLDS = {"coverage": 0.65, "overlap": 0.55}

# The refinement loop logic (compute_quality -> select strategy -> iterate) is unchanged.
```

## Best Practices

- **Do:** Tune quality thresholds (`τ_c`, `τ_o`) on a validation set rather than guessing. These are domain-sensitive — surgical images need different thresholds than satellite imagery.
- **Do:** Use hierarchical learning rates when fine-tuning SAM. Freezing the text encoder and using aggressive layer decay on the vision backbone prevents catastrophic forgetting of pre-trained features.
- **Do:** Include the presence prediction loss in training. Without it, the model hallucinates masks for objects that are not in the image, which cascades into false positives in the refinement loop.
- **Do:** Cap the maximum iteration count at 2-3. Diminishing returns kick in quickly, and excessive re-segmentation can degrade already-good regions.
- **Avoid:** Skipping the morphological post-processing even when quality passes. Noise removal and hole-filling are cheap and consistently improve mask boundaries.
- **Avoid:** Using the VLM for pixel-level assessment. VLMs are good at bounding-box-level detection but unreliable for precise boundary evaluation. The coverage/overlap metrics bridge this gap by converting VLM outputs into quantitative mask quality signals.
- **Avoid:** Treating human feedback and VLM assessment equally. Human corrections must override VLM judgments — the VLM is a proxy; the human is the authority.

## Error Handling

| Problem | Cause | Solution |
|---------|-------|----------|
| VLM returns no detections | Unusual image, occluded objects, or prompt mismatch | Fall back to trust-initial strategy (morphological cleanup only); log for review |
| Refinement loop oscillates (S flips between 0 and 1) | Thresholds too close to boundary values, or re-segmentation slightly degrades adjacent regions | Add hysteresis: once a region passes quality, lock it and do not re-evaluate |
| SAM predicts empty mask | Query describes an object not present in image | Check the presence prediction head output; if confidence is below threshold, return empty mask with a "not found" status instead of iterating |
| VLM bounding boxes are grossly inaccurate | Domain shift or ambiguous scene | Cross-check box count against expected range; if VLM returns implausible number of detections, skip VLM-guided refinement and rely on initial segmentation |
| Quality metrics are high but visual quality is poor | Metrics don't capture boundary accuracy | Add an optional boundary-quality check using edge detection or contour smoothness; flag masks with high coverage but jagged boundaries |

## Limitations

- **VLM latency dominates inference time.** Each iteration requires a VLM call for quality assessment, making this approach significantly slower than single-pass segmentation. Not suitable for real-time (<100ms) requirements without caching or distillation.
- **Bounding-box-level quality assessment has a ceiling.** The coverage and overlap metrics cannot detect fine-grained boundary errors within a correctly-located bounding box. A mask that roughly covers the right region will pass quality checks even if boundaries are imprecise.
- **Multi-granularity annotations require a well-defined taxonomy.** For novel or ambiguous domains where category hierarchies are unclear, the 3-level annotation scheme may not help or could introduce confusion.
- **The approach assumes VLM detection is reasonably accurate.** If the VLM systematically misses objects or hallucinates detections, the refinement loop amplifies those errors rather than correcting them.
- **Clinician-in-the-loop feedback requires a UI.** The pipeline architecture supports it, but building the interactive feedback interface is a separate engineering effort not covered by the model itself.

## Reference

**Paper:** Lou, A., Li, Y., Chang, Q., Xi, N., & Xie, L. (2026). *VLM-Guided Iterative Refinement for Surgical Image Segmentation with Foundation Models.* arXiv:2602.09252v1. [https://arxiv.org/abs/2602.09252v1](https://arxiv.org/abs/2602.09252v1)

**What to look for:** Section 3 details the four-stage pipeline (initial segmentation, VLM detection, quality evaluation, agentic refinement). Table 2 shows the quality metric formulations. Algorithm 1 provides the full iterative refinement pseudocode. Section 4.1 describes the multi-granularity annotation scheme and hierarchical fine-tuning strategy.