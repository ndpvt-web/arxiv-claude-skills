---
name: "yoloe-26-integrating-yolo26-yoloe"
description: |
  Build and deploy real-time open-vocabulary instance segmentation pipelines using YOLOE-26, which combines YOLOv26's NMS-free architecture with YOLOE's open-vocabulary embedding heads. Covers text-prompted, visual-prompted, and prompt-free segmentation via the Ultralytics API.
  Trigger phrases:
  - "Set up YOLOE-26 for open-vocabulary segmentation"
  - "Detect and segment arbitrary object classes in real time"
  - "Build a visual prompt segmentation pipeline with YOLO"
  - "Deploy open-vocabulary instance segmentation with TensorRT"
  - "Segment objects by text description without retraining"
  - "Use YOLOE for prompt-free object detection"
---

# YOLOE-26: Real-Time Open-Vocabulary Instance Segmentation

This skill enables Claude to build, configure, and deploy real-time open-vocabulary instance segmentation systems using YOLOE-26. Unlike closed-set detectors that only recognize fixed training classes, YOLOE-26 replaces class logits with a unified object embedding head that matches detections against text descriptions, visual examples, or a built-in 4,585-category vocabulary — all within the Ultralytics YOLO ecosystem with no NMS post-processing.

## When to Use

- When the user needs to detect and segment object classes not seen during training, specified via natural language text prompts
- When building a segmentation system that accepts visual examples (bounding boxes of reference objects) as prompts instead of text
- When deploying a fully autonomous segmentation model that discovers objects without any prompt (prompt-free mode)
- When the user wants real-time instance segmentation (30+ FPS) on edge devices via ONNX/TensorRT export
- When choosing between YOLOE-26 model sizes (n/s/m/l/x) for a specific accuracy-latency trade-off
- When integrating open-vocabulary segmentation into an existing Ultralytics training or deployment pipeline
- When the user asks to segment novel or rare object categories without collecting new training data

## Key Technique

YOLOE-26 fuses two advances: YOLOv26's NMS-free, end-to-end convolutional detection pipeline and YOLOE's open-vocabulary learning via embedding similarity. The core architectural change is replacing the final classification layer (fixed C-class logits) with an **object embedding head** that outputs D-dimensional vectors for each of N anchor points. Classification becomes a dot-product similarity: `Labels = O * P^T` where `O ∈ R^{N×D}` are anchor embeddings and `P ∈ R^{C×D}` are prompt embeddings from any source. This single change enables three prompting modes through one unified embedding space.

Three modules make this practical. **RepRTA (Re-Parameterizable Region-Text Alignment)** trains a lightweight auxiliary network to align region features with text embeddings from a language encoder, then re-parameterizes those weights into the embedding head at export time — yielding zero additional inference cost for text prompts. **SAVPE (Semantic-Activated Visual Prompt Encoder)** encodes visual examples (reference bounding boxes with class labels) into the same embedding space via semantic and activation branches, enabling few-shot segmentation from image crops. **Lazy Region-Prompt Contrast (LRPC)** provides prompt-free inference by filtering anchor points against a special prompt embedding (`O' = {o ∈ O | o · P_s^T > δ}`), then matching survivors against a built-in vocabulary of 4,585 categories derived from RAM++ tags.

Training follows a three-stage strategy: (1) text-prompt pretraining on Objects365v1, GQA, and Flickr30k with pseudo-masks from SAM, (2) visual-prompt adaptation, and (3) prompt-free specialization. The total loss combines embedding-based BCE classification, CIoU/GIoU box regression, pixel-wise BCE + dice mask loss, and a refinement term. All stages remain fully compatible with Ultralytics `train`, `val`, and `export` workflows.

## Model Size Reference

| Model             | mAP50-95 (text) | Params | FLOPs   | Use Case                        |
|-------------------|-----------------|--------|---------|----------------------------------|
| YOLOE-26n-seg     | 23.7            | 4.8M   | 6.0B    | Edge/mobile, highest FPS         |
| YOLOE-26s-seg     | 29.9            | 13.1M  | 21.7B   | Balanced edge deployment         |
| YOLOE-26m-seg     | 35.4            | 27.9M  | 70.1B   | Server-side, good accuracy       |
| YOLOE-26l-seg     | 36.8            | 32.3M  | 88.3B   | High accuracy, real-time on GPU  |
| YOLOE-26x-seg     | 39.5            | 69.9M  | 196.7B  | Maximum accuracy                 |

Prompt-free variant (YOLOE-26x-seg-pf): 29.9 mAP50-95 over 4,585 categories.

## Step-by-Step Workflow

1. **Install the Ultralytics ecosystem** with YOLOE-26 support:
   ```bash
   pip install ultralytics>=8.3.0
   ```
   Verify GPU availability with `python -c "import torch; print(torch.cuda.is_available())"`.

2. **Select model size** based on deployment target. Use `n` or `s` for edge/mobile (< 15M params), `m` or `l` for GPU servers, `x` for maximum accuracy when latency budget allows. Download weights:
   ```python
   from ultralytics import YOLO
   model = YOLO("yoloe-26l-seg.pt")  # auto-downloads from Ultralytics hub
   ```

3. **Configure the prompting mode** based on the use case:
   - **Text-prompted**: Call `model.set_classes(names, model.get_text_pe(names))` with a list of target class names.
   - **Visual-prompted**: Provide reference bounding boxes and class IDs via `visual_prompts` dict.
   - **Prompt-free**: Load the `-pf` variant (`yoloe-26l-seg-pf.pt`) — no configuration needed.

4. **Run inference** with `model.predict()`, passing image paths, directories, video streams, or numpy arrays. For visual prompts, set `predictor=YOLOEVPSegPredictor`.

5. **Process results**: Each `Results` object contains `boxes` (xyxy + confidence + class), `masks` (binary segmentation masks), and `names` (class name mapping). Iterate with `for r in results:` to access per-image outputs.

6. **Export for production deployment**:
   ```python
   model.export(format="engine", half=True)  # TensorRT FP16
   model.export(format="onnx", simplify=True)  # ONNX for cross-platform
   ```
   Note: RepRTA is already re-parameterized into the embedding head, so text-prompted models have zero overhead vs closed-set at inference.

7. **Fine-tune on domain data** (optional) using the standard Ultralytics training API with grounding-style annotations:
   ```python
   model.train(data="custom_grounding.yaml", epochs=50, imgsz=640)
   ```

8. **Validate** against LVIS or COCO benchmarks:
   ```python
   metrics = model.val(data="lvis.yaml")
   print(f"mAP50-95: {metrics.seg.map}")
   ```

9. **Deploy to a video stream** for real-time inference:
   ```python
   results = model.predict(source="rtsp://camera:554/stream", stream=True)
   for r in results:
       annotated = r.plot()
       # Display or send annotated frame
   ```

10. **Switch prompting modes at runtime** without reloading the model — call `set_classes()` to change text prompts, or pass new `visual_prompts` per frame. The unified embedding space makes mode switching a simple API call.

## Concrete Examples

**Example 1: Text-Prompted Segmentation of Custom Classes**

User: "I need to detect and segment 'forklift', 'hard hat', and 'safety vest' in warehouse surveillance footage using YOLOE-26."

Approach:
1. Load the `l`-size model for GPU server deployment
2. Set custom classes via text prompts
3. Run on video source with streaming

```python
from ultralytics import YOLO

model = YOLO("yoloe-26l-seg.pt")

# Define open-vocabulary classes — no retraining needed
classes = ["forklift", "hard hat", "safety vest"]
model.set_classes(classes, model.get_text_pe(classes))

# Stream inference on surveillance feed
for result in model.predict("warehouse_cam.mp4", stream=True, conf=0.3):
    masks = result.masks  # Binary masks for each detection
    for box, mask in zip(result.boxes, masks):
        cls_name = result.names[int(box.cls)]
        confidence = float(box.conf)
        print(f"Detected {cls_name} ({confidence:.2f})")
    result.save(filename="annotated_frame.jpg")  # Save with overlaid masks
```

Output: Annotated video frames with pixel-level segmentation masks for forklifts, hard hats, and safety vests — classes never seen during YOLOE-26 training.

---

**Example 2: Visual-Prompted Segmentation from Reference Bounding Boxes**

User: "I have a reference image where I've marked two object types with bounding boxes. Use those as visual prompts to segment similar objects in new images."

Approach:
1. Define reference bounding boxes and class assignments
2. Use the visual prompt predictor
3. Apply to new images

```python
import numpy as np
from ultralytics import YOLO
from ultralytics.models.yolo.yoloe import YOLOEVPSegPredictor

model = YOLO("yoloe-26l-seg.pt")

# Reference bounding boxes from the example image [x1, y1, x2, y2]
visual_prompts = dict(
    bboxes=np.array([
        [120.0, 200.5, 340.0, 580.0],   # Example of class 0 (e.g., custom part A)
        [450.0, 100.0, 520.0, 250.0],   # Example of class 1 (e.g., custom part B)
    ]),
    cls=np.array([0, 1])
)

# Segment similar objects in a new image using visual examples
results = model.predict(
    "new_image.jpg",
    visual_prompts=visual_prompts,
    predictor=YOLOEVPSegPredictor,
    conf=0.25
)
results[0].show()
```

Output: Instance segmentation masks on the new image for objects visually similar to the reference bounding box crops, without any text description needed.

---

**Example 3: Prompt-Free Autonomous Segmentation and TensorRT Export**

User: "Deploy a model that automatically discovers and segments all objects in a scene — no prompts, no predefined classes. Export it for TensorRT."

Approach:
1. Load the prompt-free variant
2. Export to TensorRT FP16
3. Run inference with the exported engine

```python
from ultralytics import YOLO

# Load prompt-free model (built-in 4,585-category vocabulary)
model = YOLO("yoloe-26l-seg-pf.pt")

# Export to TensorRT for production
model.export(format="engine", half=True, imgsz=640)

# Load exported engine and run
engine_model = YOLO("yoloe-26l-seg-pf.engine")
results = engine_model.predict("scene.jpg", conf=0.3)

# The model autonomously identifies objects from 4,585 categories
for box in results[0].boxes:
    cls_id = int(box.cls)
    cls_name = results[0].names[cls_id]
    print(f"Discovered: {cls_name} (conf: {float(box.conf):.2f})")
```

Output: All recognizable objects segmented and labeled from RAM++ vocabulary — no user input required. TensorRT engine provides maximum throughput for deployment.

## Best Practices

- **Do** start with the `l`-size model for prototyping — it offers the best accuracy-to-cost ratio (36.8 mAP at 32.3M params) before deciding if you need `x` or can drop to `s`.
- **Do** use `stream=True` for video inference to avoid loading all frames into memory.
- **Do** set confidence thresholds (`conf=0.25` to `0.4`) appropriate to your domain — open-vocabulary models produce more candidate detections than closed-set models.
- **Do** re-parameterize (via `model.export()`) before deployment — RepRTA folds into the embedding head, eliminating the text encoder from the inference graph.
- **Avoid** using prompt-free mode when you know the target classes — text-prompted mode is significantly more accurate (39.5 vs 29.9 mAP on the `x` model) because it narrows the search space.
- **Avoid** mixing visual and text prompts in the same inference call — choose one modality per prediction. Switch between them across frames as needed.
- **Avoid** using the `n` variant for open-vocabulary tasks requiring fine-grained distinctions — at 23.7 mAP, it is best suited for detecting large, well-separated objects on constrained hardware.

## Error Handling

- **Low confidence on novel classes**: Open-vocabulary models may score novel categories lower than training-domain objects. Lower `conf` threshold to 0.15-0.2 and filter results post-hoc if recall matters more than precision.
- **Text prompt wording sensitivity**: Embedding similarity depends on prompt phrasing. If "car" misses detections, try "automobile", "vehicle", or "sedan". Use the most common noun form.
- **Visual prompt quality**: SAVPE relies on representative reference crops. Ensure bounding boxes tightly enclose the reference object and the reference image has similar lighting/angle to target images.
- **Export failures**: TensorRT export requires matching CUDA/TensorRT versions. If export fails, fall back to ONNX (`format="onnx"`) which is more portable.
- **Out-of-memory on large images**: The model expects 640px input by default. For high-resolution images, use `imgsz=640` (default) and let Ultralytics handle resizing, or tile the image and merge results.
- **Prompt-free vocabulary gaps**: The built-in 4,585 categories from RAM++ may not cover highly specialized domains (e.g., medical imaging, microscopy). Use text-prompted mode with domain-specific class names instead.

## Limitations

- Open-vocabulary performance (23.7-39.5 mAP) is lower than closed-set YOLO models trained on the same categories, due to the generalization-accuracy trade-off inherent in embedding-based classification.
- Visual prompting (SAVPE) requires at least one reference bounding box per target class and is sensitive to domain shift between reference and target images.
- The built-in prompt-free vocabulary is English-centric and biased toward common everyday objects; niche industrial or scientific categories may be absent or poorly represented.
- Real-time performance claims assume GPU inference (e.g., T4/A100); CPU-only deployment with larger models will not achieve real-time rates.
- The model does not support open-vocabulary panoptic segmentation — it handles instance segmentation (countable objects) but not stuff classes (sky, road, grass).

## Reference

**Paper**: [YOLOE-26: Integrating YOLO26 with YOLOE for Real-Time Open-Vocabulary Instance Segmentation](https://arxiv.org/abs/2602.00168v1) — Sapkota & Karkee, 2026. Focus on Section 3 (architecture: RepRTA, SAVPE, LRPC), Table 1 (model scaling results), and Section 4 (training strategy and dataset composition).