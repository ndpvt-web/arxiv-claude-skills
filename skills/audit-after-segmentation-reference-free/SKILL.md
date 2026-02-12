---
name: "audit-after-segmentation-reference-free"
description: "Build reference-free mask quality assessment pipelines for multimodal segmentation systems. Implements the MQ-Auditor pattern: given a segmentation mask and multimodal context (video/audio/text), estimate IoU without ground truth, classify error type (geometric vs semantic), and recommend accept/refine/reject actions. Use when: 'audit segmentation masks without ground truth', 'build a mask quality checker', 'add reference-free QA to my segmentation pipeline', 'classify segmentation errors automatically', 'detect segmentation failures in video', 'create a reflection agent for mask refinement'."
---

# Audit After Segmentation: Reference-Free Mask Quality Assessment

This skill enables Claude to design and implement **reference-free mask quality assessment (MQA) systems** for segmentation pipelines. Based on the MQ-Auditor approach from Zhou et al. (2026), the core idea is to evaluate segmentation mask quality *without* ground-truth annotations at inference time by reasoning over multimodal context. The system produces three outputs per mask: a predicted IoU score, an error type classification, and an actionable quality-control decision (accept / refine / reject). This transforms segmentation from a fire-and-forget operation into an auditable pipeline with built-in failure detection.

## When to Use

- When building a segmentation pipeline that needs automated quality control without human annotation
- When the user wants to detect and classify segmentation failures (missed regions, boundary errors, wrong objects)
- When integrating a "reflection agent" that can diagnose mask quality and trigger re-segmentation
- When constructing a benchmark of diverse mask error modes for training or evaluation
- When the user needs to estimate IoU between a predicted mask and unknown ground truth
- When building multimodal systems that must jointly reason over video, audio, text, and spatial masks
- When the user asks to add self-correction or quality gating to an existing vision model

## Key Technique

### The Three-Output Assessment Framework

Traditional segmentation evaluation requires ground-truth masks (IoU, Dice, etc.). MQ-Auditor eliminates this dependency by training an MLLM to predict mask quality from multimodal context alone. Given a video frame, synchronized audio, a referring text expression, and a candidate mask, the model produces: **(1)** a continuous IoU estimate in [0, 1], **(2)** an error type from a six-class taxonomy, and **(3)** a recommended action. The error taxonomy spans **geometric errors** (cutout: interior holes; dilate: boundary over-expansion; erode: boundary under-coverage) and **semantic errors** (perfect: correct mask; full_neg: entirely wrong object; merge: target plus distractors). Actions map naturally: perfect -> accept, geometric errors -> refine, full_neg -> reject, merge -> refine or reject.

### Dual Mask Representation

The critical architectural insight is how mask information is encoded. Rather than passing the binary mask alone, the system creates two complementary representations: (a) a **masked frame** (element-wise multiplication of the video frame and mask, showing *what* the mask contains semantically) and (b) the **binary mask as pseudo-RGB** (showing *where* the mask is spatially). These are concatenated and fed alongside audio/text embeddings into the LLM backbone. This dual representation lets the model reason about both "does this mask cover the right object?" and "does this mask have the right shape?"

### Integration as a Reflection Agent

The most powerful application is using MQA as a reflection loop: a baseline segmenter produces masks -> the auditor evaluates them -> failures (especially full_neg) are caught -> corrected target descriptions are extracted from the auditor's reasoning -> a grounding model (e.g., Grounded-SAM2) re-segments using corrected guidance. This closed loop showed 40% Jaccard improvement on unseen categories.

## Step-by-Step Workflow

1. **Define your mask representation pipeline.** For each candidate mask, generate two tensors: (a) the masked frame (`frame * mask`) preserving RGB content only within the mask region, and (b) the binary mask converted to a 3-channel pseudo-RGB image (white on black). Concatenate these along the channel or spatial dimension as model input.

2. **Encode multimodal context.** Process the video frame through a vision encoder (e.g., CLIP ViT-L/14), audio through an audio encoder (e.g., BEATs), and the referring text through the LLM's tokenizer. Use Q-Former modules with learnable query tokens (32 per modality) plus linear projectors to align each modality's embeddings into the LLM's input space.

3. **Design the error taxonomy for your domain.** Adapt the six-class taxonomy to your use case:
   - **Perfect**: IoU ~ 1.0 -> action: accept
   - **Cutout**: interior regions missing -> action: refine
   - **Dilate**: boundaries over-expanded -> action: refine
   - **Erode**: boundaries under-segmented -> action: refine
   - **Merge**: correct target + extra objects -> action: refine or reject
   - **Full_neg**: entirely wrong object, IoU ~ 0 -> action: reject

4. **Construct training data with controlled error modes.** Generate geometric variants from ground-truth masks using morphological operations (OpenCV erode/dilate with calibrated kernel sizes to hit target IoU ranges like [0.85, 0.95] for easy and [0.5, 0.7] for hard). Generate semantic negatives by using a VLM to propose distractor objects, a grounding model to localize them, and SAM2 to produce masks. Retain the top-3 hardest negatives by bounding-box IoU overlap with ground truth.

5. **Structure the training prompt.** Use an instruction-tuning format with a system prompt defining the auditor role, followed by multimodal tokens (video, audio, mask representations), the referring expression, and the question: "Estimate the IoU, identify the error type, and recommend an action." Train the model to output all three predictions as structured text via next-token prediction.

6. **Train with balanced sampling.** Use a 50% positive (perfect mask) ratio per mini-batch to prevent the model from becoming biased toward predicting errors. Apply LoRA (rank 32, alpha 64) on the LLM backbone with AdamW at lr=1e-4 and bf16 precision. This balancing is critical -- ablations show 50% positive ratio gives the best trade-off between IoU RMSE and classification F2-score.

7. **Evaluate with recall-weighted metrics.** Use RMSE for IoU prediction accuracy and F2-score (not F1) for error type and action classification. F2 emphasizes recall because missing a bad mask (false negative) is more costly than flagging a good mask (false positive) in quality-control contexts.

8. **Integrate as a reflection loop.** Wire the auditor into your pipeline: run baseline segmentation -> audit each mask -> if action is "reject," extract the auditor's reasoning about what went wrong -> re-prompt a grounding segmenter (e.g., Grounded-SAM2) with corrected target description -> re-audit the new mask. Cap the loop at 2-3 iterations to avoid infinite cycles.

9. **Implement action-gated output.** In production, use the three-way action as a routing signal: "accept" passes the mask through, "refine" triggers post-processing (CRF, boundary refinement, or re-segmentation), "reject" flags the sample for human review or alternative processing.

## Concrete Examples

**Example 1: Adding quality gating to a video segmentation API**

User: "I have a Ref-AVS model that segments objects described by text in video. Sometimes it completely misses the target or grabs the wrong object. I want to add automatic quality checking."

Approach:
1. After the segmenter produces a mask for each frame, generate the dual mask representation (masked frame + binary pseudo-RGB).
2. Feed the frame, audio spectrogram, referring text, and dual mask into a quality assessment model.
3. Parse the three outputs: IoU estimate, error type, action.
4. Route based on action:

```python
class MaskQualityGate:
    def __init__(self, auditor_model, refiner_model, max_retries=2):
        self.auditor = auditor_model
        self.refiner = refiner_model
        self.max_retries = max_retries

    def assess_and_route(self, frame, audio, text, mask):
        """Assess mask quality and route to appropriate action."""
        # Create dual mask representation
        masked_frame = frame * mask[..., None]  # (H, W, 3)
        binary_rgb = np.stack([mask * 255] * 3, axis=-1)  # pseudo-RGB

        # Run auditor
        iou_est, error_type, action = self.auditor.predict(
            frame=frame, audio=audio, text=text,
            masked_frame=masked_frame, binary_mask=binary_rgb
        )

        if action == "accept":
            return mask, {"iou": iou_est, "status": "accepted"}
        elif action == "refine":
            refined = self.refiner.refine(frame, mask, error_type)
            return refined, {"iou": iou_est, "error": error_type, "status": "refined"}
        else:  # reject
            for attempt in range(self.max_retries):
                new_mask = self.refiner.re_segment(frame, audio, text)
                iou_est, error_type, action = self.auditor.predict(
                    frame=frame, audio=audio, text=text,
                    masked_frame=frame * new_mask[..., None],
                    binary_mask=np.stack([new_mask * 255] * 3, axis=-1)
                )
                if action != "reject":
                    return new_mask, {"iou": iou_est, "status": "re-segmented"}
            return None, {"status": "failed", "error": error_type}
```

**Example 2: Generating a mask error benchmark from existing annotations**

User: "I have ground-truth segmentation masks. I want to create a training dataset with diverse error types for a quality assessment model."

Approach:
1. Generate geometric variants via morphological operations.
2. Generate semantic negatives via VLM + grounding model.
3. Label each with IoU, error type, and recommended action.

```python
import cv2
import numpy as np

def generate_geometric_errors(gt_mask, target_iou_ranges):
    """Generate cutout, dilate, erode variants from ground truth."""
    variants = []

    # Erode: shrink mask boundaries
    for kernel_size in range(3, 25, 2):
        kernel = cv2.getStructuringElement(cv2.MORPH_ELLIPSE, (kernel_size, kernel_size))
        eroded = cv2.erode(gt_mask, kernel, iterations=1)
        iou = compute_iou(gt_mask, eroded)
        variants.append({"mask": eroded, "iou": iou, "type": "erode", "action": "refine"})

    # Dilate: expand mask boundaries
    for kernel_size in range(3, 25, 2):
        kernel = cv2.getStructuringElement(cv2.MORPH_ELLIPSE, (kernel_size, kernel_size))
        dilated = cv2.dilate(gt_mask, kernel, iterations=1)
        iou = compute_iou(gt_mask, dilated)
        variants.append({"mask": dilated, "iou": iou, "type": "dilate", "action": "refine"})

    # Cutout: remove random interior regions
    for _ in range(10):
        cutout = gt_mask.copy()
        ys, xs = np.where(gt_mask > 0)
        if len(ys) == 0:
            continue
        cy, cx = np.random.choice(ys), np.random.choice(xs)
        radius = np.random.randint(5, max(6, int(np.sqrt(gt_mask.sum()) * 0.3)))
        cv2.circle(cutout, (cx, cy), radius, 0, -1)
        iou = compute_iou(gt_mask, cutout)
        variants.append({"mask": cutout, "iou": iou, "type": "cutout", "action": "refine"})

    # Filter to target IoU ranges
    filtered = []
    for low, high in target_iou_ranges:
        matches = [v for v in variants if low <= v["iou"] <= high]
        if matches:
            filtered.append(min(matches, key=lambda v: abs(v["iou"] - (low+high)/2)))
    return filtered

def generate_semantic_negatives(frame, gt_mask, text, vlm, grounder, sam):
    """Generate full_neg and merge masks using VLM-guided pipeline."""
    # Ask VLM for distractor objects (descriptive noun phrases, not categories)
    distractors = vlm.generate(
        frame=frame,
        prompt=f"List 5 objects visible in this image that are NOT '{text}'. "
               f"Use descriptive noun phrases like 'red wooden chair' not just 'chair'."
    )
    negatives = []
    for desc in distractors:
        boxes = grounder.detect(frame, desc)  # e.g., Grounding-DINO
        for box in boxes:
            neg_mask = sam.segment(frame, box)
            bb_iou = bbox_iou(mask_to_bbox(neg_mask), mask_to_bbox(gt_mask))
            negatives.append({"mask": neg_mask, "desc": desc, "bb_iou": bb_iou})

    # Keep top-3 hardest negatives (highest bbox overlap with GT)
    negatives.sort(key=lambda x: x["bb_iou"], reverse=True)
    full_negs = [{"mask": n["mask"], "iou": 0.0, "type": "full_neg", "action": "reject"}
                 for n in negatives[:3]]

    # Merge: union of GT + distractor
    merges = [{"mask": np.clip(gt_mask + n["mask"], 0, 1),
               "iou": compute_iou(gt_mask, np.clip(gt_mask + n["mask"], 0, 1)),
               "type": "merge", "action": "refine"}
              for n in negatives[:3]]

    return full_negs + merges
```

**Example 3: Structured quality report for a batch of masks**

User: "I want to generate a quality report summarizing segmentation results across a video dataset."

Approach:
1. Run the auditor on all masks in the dataset.
2. Aggregate statistics by error type and action.
3. Output a structured report.

```
Output:
=== Segmentation Quality Audit Report ===
Dataset: RefAVS-val (534 videos, 5340 frames)

Overall IoU Distribution:
  Mean predicted IoU: 0.72 (std: 0.28)
  Median: 0.81

Error Type Breakdown:
  perfect   : 187 (35.0%)  -> all accepted
  cutout    :  64 (12.0%)  -> mean IoU 0.78, all refined
  dilate    :  53  (9.9%)  -> mean IoU 0.74, all refined
  erode     :  41  (7.7%)  -> mean IoU 0.82, all refined
  merge     :  89 (16.7%)  -> mean IoU 0.55, 67 refined / 22 rejected
  full_neg  : 100 (18.7%)  -> mean IoU 0.03, all rejected

Action Summary:
  accept :  187 (35.0%)  -- pass through
  refine :  225 (42.1%)  -- trigger boundary/region correction
  reject :  122 (22.9%)  -- flag for re-segmentation or human review

High-Risk Videos (>50% reject rate):
  video_0042, video_0189, video_0301 -- recommend manual inspection
```

## Best Practices

- **Do:** Use the dual mask representation (masked frame + binary pseudo-RGB). The masked frame captures semantic content ("what is segmented?") while the binary mask captures geometry ("where is the boundary?"). Dropping either degrades performance significantly.
- **Do:** Balance positive (perfect) and negative (error) samples at a 50/50 ratio during training. Imbalanced training causes the model to default-predict either "accept" or "reject."
- **Do:** Use F2-score (recall-weighted) rather than F1 for evaluation. In quality control, missing a bad mask costs more than falsely flagging a good one.
- **Do:** Generate hard negatives -- distractor objects that spatially overlap with the target's bounding box. Easy negatives (objects on the other side of the frame) don't teach the model much.
- **Avoid:** Relying on IoU estimation alone. The error type classification provides actionable signal that pure IoU cannot -- a 0.6 IoU from erosion needs different correction than a 0.6 IoU from merging.
- **Avoid:** Running the reflection loop indefinitely. Cap re-segmentation attempts at 2-3 iterations. If the auditor keeps rejecting, escalate to human review.

## Error Handling

- **Auditor disagrees with visual evidence:** When the IoU estimate seems wrong, check whether the masked frame representation was generated correctly (common bug: forgetting to broadcast the mask to 3 channels before multiplying with the frame).
- **All masks classified as one type:** This almost always indicates training data imbalance. Verify that the positive sampling ratio is 50% and that all six error types are represented in training batches.
- **Reflection loop produces worse masks:** The re-segmentation step depends on extracting a corrected target description from the auditor's reasoning. If the auditor's text output is vague or incorrect, the grounding model will also fail. Add a confidence threshold on the auditor's IoU estimate before triggering re-segmentation.
- **Audio modality unavailable:** The framework degrades gracefully without audio for many visual-only tasks. However, for audio-referred objects (e.g., "the instrument being played"), audio is critical for disambiguating the correct target. Fall back to text-only grounding if audio is absent.

## Limitations

- The approach requires an MLLM backbone (7B+ parameters), making it impractical for edge deployment or real-time inference on resource-constrained hardware.
- IoU estimation accuracy degrades for masks with IoU in the 0.3-0.7 range (the "ambiguous zone"), where geometric and semantic errors overlap and are hardest to distinguish.
- The six-class error taxonomy covers common failure modes but does not handle compound errors (e.g., a mask that is both eroded AND merged) -- each mask is assigned exactly one type.
- Performance on entirely unseen object categories or domains (e.g., medical imaging) requires domain-specific fine-tuning of both the error generation pipeline and the auditor.
- The reflection loop depends on external models (VLM for descriptions, grounding model for localization, SAM for re-segmentation), making the full pipeline complex to deploy and debug.

## Reference

**Paper:** Zhou et al., "Audit After Segmentation: Reference-Free Mask Quality Assessment for Language-Referred Audio-Visual Segmentation" (arXiv:2602.03892, 2026). Look for: the dual mask representation design (Sec. 3), the six-class error taxonomy (Sec. 4), balanced training ablations (Sec. 5), and the reflection-agent integration results showing 40% Jaccard improvement (Sec. 5.4). Code: https://github.com/jasongief/MQA-RefAVS