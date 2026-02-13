---
name: annotation-free-spacecraft-detection
description: >
  Build annotation-free object detection and segmentation pipelines using VLM pseudo-labeling
  and teacher-student distillation, based on the spacecraft detection method from arXiv:2602.04699.
  Applies to any domain where manual annotation is expensive or infeasible.
  Trigger phrases: "detect objects without labels", "annotation-free detection",
  "pseudo-label pipeline", "VLM zero-shot to trained model", "teacher-student distillation
  from VLM", "spacecraft detection and segmentation"
---

# Annotation-Free Object Detection via VLM Pseudo-Labeling and Distillation

This skill enables Claude to help users build **annotation-free detection and segmentation pipelines** that eliminate the need for manual labeling. The core technique uses a large Vision Language Model (Grounded SAM-2) to generate pseudo-labels on unlabeled images via text prompts, refines those labels with test-time augmentation and weighted box fusion, then distills the knowledge into a lightweight model (YOLOv11 or EfficientDet) through iterative teacher-student training. Originally developed for spacecraft detection in orbit imagery, the pattern generalizes to any domain where annotation is costly — medical imaging, underwater robotics, satellite Earth observation, industrial inspection, and more.

## When to Use

- When the user needs to **detect or segment objects but has no labeled training data** and wants to avoid manual annotation
- When the user asks how to go from a **zero-shot VLM to a deployable lightweight detector** (e.g., "I have unlabeled images, how do I train YOLO on them?")
- When the user is working with **space imagery** (spacecraft, satellites, orbital debris) and needs detection/segmentation
- When the user wants to **generate pseudo-labels** from a foundation model like Grounded SAM-2, Grounding DINO, or Florence-2
- When the user asks about **teacher-student distillation** to compress a large VLM into a fast inference model
- When the user needs to handle **domain-specific imagery** with challenging conditions (low light, high contrast, cluttered backgrounds) where manual annotation is unreliable
- When the user asks about **test-time augmentation strategies** for improving pseudo-label quality

## Key Technique

The pipeline solves a fundamental problem: large VLMs like Grounded SAM-2 can detect objects zero-shot via text prompts, but they are too slow and resource-heavy for deployment. Meanwhile, fast models like YOLOv11 need labeled data that doesn't exist. The solution is a four-stage bridge: (1) use the VLM to auto-label a small subset of images, (2) refine those noisy labels, (3) train a lightweight model on the refined pseudo-labels, and (4) use that trained model's predictions to relabel the data and retrain — an iterative distillation loop.

The critical insight is that **even noisy pseudo-labels, when filtered and refined, produce student models that outperform the original VLM's zero-shot predictions by up to 10 AP points**. This happens because the student model learns to generalize across the dataset rather than making independent per-image predictions. The refinement stage uses vertical-flip test-time augmentation (TTA) combined with Weighted Box Fusion (WBF) at IoU threshold 0.55, which consistently outperforms adding more augmentation types — more augmentations introduce low-confidence noise that degrades label quality.

The distillation loss combines classification and regression terms: `L_distill = alpha * L_cls(p, y_hat) + beta * L_reg(b, b_hat)`, where the teacher's predictions serve as soft targets. After one round of distillation, the student's predictions on the training set replace the original pseudo-labels, and the student retrains on its own improved labels — a single iteration of self-distillation that yields consistent further gains.

## Step-by-Step Workflow

1. **Select and prepare unlabeled images.** Sample 500 representative images from your unlabeled dataset. Resize to 640x640 pixels. Ensure diversity across lighting conditions, viewpoints, and backgrounds. Store in a flat directory structure.

2. **Choose a text prompt for the VLM.** Craft a single-class or multi-class text prompt for Grounded SAM-2. Use the most general term for your target object (e.g., `"spacecraft"` not `"Soyuz capsule"`). For multi-class, use period-separated prompts: `"spacecraft . debris . antenna"`. Keep prompts short — VLM grounding performance degrades with verbose descriptions.

3. **Generate initial pseudo-labels with Grounded SAM-2.** Run `grounded_sam2_pseudo_label.py` (or equivalent) on your image subset. Configure the box threshold (start at 0.3) and text threshold (start at 0.25). Output COCO-format JSON annotations with both bounding boxes and segmentation masks.

   ```python
   # Core pseudo-label generation call
   from grounded_sam2 import GroundedSAM2Pipeline
   pipeline = GroundedSAM2Pipeline(
       grounding_model="GroundingDINO",
       sam2_checkpoint="sam2_hiera_large",
       box_threshold=0.3,
       text_threshold=0.25
   )
   annotations = pipeline.predict(image_dir="./unlabeled/", prompt="spacecraft")
   annotations.save_coco_json("pseudo_labels_raw.json")
   ```

4. **Refine labels with test-time augmentation and WBF.** Apply **vertical flip only** as TTA (horizontal flip and color jitter hurt performance in practice). For each image, run inference on both the original and flipped version, then fuse predictions using Weighted Box Fusion with IoU threshold `tau=0.55`. This reduces false positives and tightens bounding boxes.

   ```python
   # TTA + WBF refinement
   from ensemble_boxes import weighted_boxes_fusion
   # For each image: predict on original + vertical flip
   # Collect all boxes, scores, labels
   boxes_list = [original_boxes, flipped_boxes_unflipped]
   scores_list = [original_scores, flipped_scores]
   labels_list = [original_labels, flipped_labels]
   fused_boxes, fused_scores, fused_labels = weighted_boxes_fusion(
       boxes_list, scores_list, labels_list, iou_thr=0.55
   )
   ```

5. **Filter by confidence threshold.** Remove all predictions below a confidence threshold — use 0.5 as default, increase to 0.6 for cleaner domains. This step typically retains 80-95% of images (e.g., 395-471 out of 500 in the paper's experiments). Save filtered results as a clean COCO JSON annotation file.

6. **Prepare the distillation training dataset.** Convert the filtered pseudo-labels into the format required by your student model. For YOLOv11: YOLO-format `.txt` files with normalized coordinates. For EfficientDet: COCO JSON. Split into train/val (e.g., 80/20). The key: treat pseudo-labels as ground truth for training.

7. **Train the lightweight student model.** Train YOLOv11-seg (or EfficientDet) on the pseudo-labeled dataset for 300 epochs at 640x640 resolution. Use default hyperparameters from the official implementation — the paper found no benefit from extensive tuning.

   ```bash
   # YOLOv11 training
   yolo segment train model=yolo11m-seg.pt \
       data=spacecraft_pseudo.yaml \
       epochs=300 imgsz=640 batch=16
   ```

8. **Iterative relabeling (one round).** Run the trained student model on the same training images. Replace the original pseudo-labels with the student's predictions (keeping only predictions above the confidence threshold). Retrain the student on these improved labels. One iteration is sufficient — additional rounds show diminishing returns.

9. **Evaluate and deploy.** Evaluate the final student model on a held-out test set using COCO AP metrics (AP, AP50, AP75). The student should outperform the original VLM's zero-shot predictions. Deploy the ~7M parameter model at >60 FPS for real-time inference.

## Concrete Examples

**Example 1: Spacecraft detection on unlabeled orbital imagery**

User: "I have 2000 unlabeled images of spacecraft in orbit from the SPARK dataset. I need a real-time detector but can't afford to annotate them. How do I train one?"

Approach:
1. Randomly sample 500 images from the 2000-image set
2. Run Grounded SAM-2 with prompt `"spacecraft"`, box_threshold=0.3
3. Apply vertical-flip TTA + WBF (tau=0.55) to refine detections
4. Filter predictions at confidence >= 0.5 (expect ~395 images retained)
5. Convert to YOLO format and train YOLOv11m-seg for 300 epochs at 640x640
6. Relabel training data with the student model, retrain once
7. Run final model on all 2000 images at >60 FPS

Output:
```
Model: YOLOv11m-seg (7M params)
Expected AP: ~76 (detection), ~70 (segmentation)
Inference speed: >60 FPS on GPU
Labels required: 0 manual annotations
```

**Example 2: Adapting the pipeline for underwater marine debris detection**

User: "I want to detect marine debris in underwater ROV footage without labeling data. Can I use VLM pseudo-labeling?"

Approach:
1. Extract 500 representative frames from ROV video at diverse depths/lighting
2. Resize to 640x640 and run Grounded SAM-2 with prompt `"marine debris . plastic bag . bottle . fishing net"`
3. Apply vertical-flip TTA + WBF fusion (tau=0.55) — underwater scenes benefit from the same minimal-augmentation strategy since additional transforms introduce noise in murky conditions
4. Filter at confidence >= 0.5 to remove hallucinated detections in turbid water
5. Train YOLOv11m-seg on the filtered pseudo-labels for 300 epochs
6. Run one round of iterative relabeling and retrain
7. Deploy on ROV compute hardware

Output:
```
Pipeline adapts directly — change only the text prompt and input images.
The VLM handles domain shift; distillation handles deployment constraints.
Expect 5-15 AP improvement over raw VLM zero-shot predictions.
```

**Example 3: Writing the pseudo-label generation script from scratch**

User: "Help me write a Python script that generates COCO-format pseudo-labels from Grounded SAM-2 on a folder of images."

Approach:
1. Set up Grounded SAM-2 with GroundingDINO backbone and SAM2 segmentation head
2. Iterate over images, run detection with text prompt, extract boxes + masks
3. Apply vertical-flip TTA: flip image, run inference, unflip predictions, fuse with WBF
4. Filter by confidence threshold and convert to COCO JSON format

Output:
```python
import os, json, cv2, numpy as np
from groundingdino.util.inference import load_model, predict
from sam2.build_sam import build_sam2
from ensemble_boxes import weighted_boxes_fusion

def generate_pseudo_labels(image_dir, prompt, output_json,
                           box_thr=0.3, text_thr=0.25,
                           conf_thr=0.5, wbf_iou=0.55):
    coco = {"images": [], "annotations": [], "categories": [
        {"id": 1, "name": prompt}
    ]}
    ann_id = 0

    for img_id, fname in enumerate(sorted(os.listdir(image_dir))):
        img = cv2.imread(os.path.join(image_dir, fname))
        img_rgb = cv2.cvtColor(img, cv2.COLOR_BGR2RGB)
        h, w = img.shape[:2]

        # Original prediction
        boxes_orig, scores_orig = run_grounded_sam2(img_rgb, prompt, box_thr, text_thr)
        # Vertical flip TTA
        img_flip = cv2.flip(img_rgb, 0)
        boxes_flip, scores_flip = run_grounded_sam2(img_flip, prompt, box_thr, text_thr)
        boxes_flip = unflip_vertical(boxes_flip, h)

        # Weighted Box Fusion
        boxes_norm = [normalize_boxes(boxes_orig, w, h),
                      normalize_boxes(boxes_flip, w, h)]
        scores = [scores_orig, scores_flip]
        labels = [np.zeros(len(scores_orig)), np.zeros(len(scores_flip))]
        fused_boxes, fused_scores, _ = weighted_boxes_fusion(
            boxes_norm, scores, labels, iou_thr=wbf_iou)

        # Confidence filter and write COCO annotations
        for box, score in zip(fused_boxes, fused_scores):
            if score < conf_thr:
                continue
            x1, y1, x2, y2 = box * np.array([w, h, w, h])
            coco["annotations"].append({
                "id": ann_id, "image_id": img_id,
                "category_id": 1, "score": float(score),
                "bbox": [float(x1), float(y1), float(x2-x1), float(y2-y1)],
                "area": float((x2-x1)*(y2-y1)), "iscrowd": 0
            })
            ann_id += 1

        coco["images"].append({"id": img_id, "file_name": fname,
                                "width": w, "height": h})

    with open(output_json, "w") as f:
        json.dump(coco, f)
    print(f"Generated {ann_id} pseudo-labels for {len(coco['images'])} images")
```

## Best Practices

- **Do** use the most general object-class noun as your VLM prompt (e.g., `"spacecraft"` not `"satellite module with solar panels"`). Grounding DINO performs better with concise terms.
- **Do** limit TTA to vertical flip only. The paper empirically shows that adding horizontal flip, color jitter, or brightness perturbations introduces low-confidence noise that degrades WBF output.
- **Do** use Weighted Box Fusion (WBF) instead of standard NMS for merging TTA predictions. WBF averages overlapping boxes rather than discarding all but the top-scoring one, producing tighter localizations.
- **Do** perform exactly one round of iterative relabeling. The student relabels the training data with its own improved predictions, then retrains. Multiple rounds show diminishing returns.
- **Avoid** using all available unlabeled data for pseudo-labeling. A curated subset of ~500 diverse images is sufficient and reduces noise propagation. More data with noisy labels does not help.
- **Avoid** lowering the confidence filter threshold below 0.5 to retain more pseudo-labels. The paper shows that precision matters more than recall at the pseudo-labeling stage — noisy labels hurt the student more than missing labels.

## Error Handling

| Problem | Symptom | Solution |
|---------|---------|----------|
| VLM detects nothing | 0 pseudo-labels generated | Lower `box_threshold` to 0.2 and `text_threshold` to 0.15. Try alternative prompts (e.g., `"object"` or `"satellite"` instead of `"spacecraft"`). |
| Too many false positives | Pseudo-labels contain background regions | Raise confidence filter to 0.6-0.7. Inspect worst-scoring 10% of predictions visually before training. |
| Student model diverges | Training loss spikes or NaN | Reduce learning rate by 10x. Check that pseudo-label bounding boxes are properly normalized (0-1 range for YOLO format). |
| WBF produces duplicate boxes | Multiple overlapping detections per object | Increase WBF `iou_thr` from 0.55 to 0.65. Ensure TTA predictions are properly unflipped before fusion. |
| Poor segmentation masks | Masks leak into background | SAM2 mask quality depends on box prompt quality. Tighten bounding boxes by increasing `box_threshold`. Consider using SAM2's multi-mask output and selecting the highest-confidence mask. |
| Domain gap from space to new domain | Student AP much lower than expected | The text prompt may not ground well in your domain. Test the VLM on 10 images manually before running the full pipeline. Consider using Florence-2 or a domain-specific VLM instead of Grounding DINO. |

## Limitations

- **Single-class bias**: The pipeline works best for single-class detection (one object type per prompt). Multi-class scenarios require separate prompt runs and careful category merging, which can introduce cross-class confusion.
- **Requires at least some detectable signal**: If the VLM cannot detect the target at all zero-shot (AP < 5), there is insufficient signal for distillation. The technique amplifies weak but present VLM capability — it cannot create capability from nothing.
- **Small object detection**: Grounded SAM-2 struggles with objects smaller than ~32x32 pixels at 640x640 resolution. For tiny objects, consider running at higher resolution during pseudo-labeling even if the student trains at 640x640.
- **No explicit handling of class imbalance**: If the target object appears in only 10% of images, pseudo-labeling will produce a heavily imbalanced dataset. Standard oversampling or focal loss adjustments in the student training may be needed.
- **Segmentation quality ceiling**: Mask quality is bounded by SAM2's zero-shot capability. For domains far from SAM2's training distribution (e.g., X-ray imagery), mask pseudo-labels may be too noisy to be useful even after filtering.
- **Compute cost of pseudo-labeling**: Grounded SAM-2 requires a GPU with >= 16GB VRAM for the large checkpoint. The pseudo-labeling stage itself is a one-time cost (~2 seconds per image on an A100), but it is not feasible on CPU-only machines.

## Reference

**Paper**: [Annotation Free Spacecraft Detection and Segmentation using Vision Language Models](https://arxiv.org/abs/2602.04699v1) (ICRA 2026)
**Code**: [github.com/giddyyupp/annotation-free-spacecraft-segmentation](https://github.com/giddyyupp/annotation-free-spacecraft-segmentation)
**Key finding**: A VLM pseudo-label + single-round teacher-student distillation pipeline yields lightweight models (7M params, >60 FPS) that outperform the source VLM's zero-shot inference by up to 10 AP points on spacecraft segmentation, with zero manual annotations.