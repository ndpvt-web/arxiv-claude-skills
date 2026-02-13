---
name: "revisiting-salient-object-detection"
description: "Build observer-centric salient object detection systems using the Perceive-Reflect-Adjust agentic loop. Combines a Vision-Language Model with SAM2 segmentation, conditioning on observer intent or preference to produce personalized saliency maps. Trigger phrases: 'detect salient objects for a specific user persona', 'observer-centric saliency', 'personalized object segmentation', 'perceive-reflect-adjust pipeline', 'intent-driven saliency detection', 'build an OC-SOD agent'."
---

# Observer-Centric Salient Object Detection (OC-SOD)

This skill enables Claude to design and implement observer-centric salient object detection systems that move beyond single-ground-truth saliency maps. Instead of treating saliency as an objective property of an image, this approach conditions detection on **who is looking** -- their preferences, intent, or persona -- using a VLM + SAM2 agentic loop called "Perceive-Reflect-Adjust." The result is a system that produces different, valid saliency masks for the same image depending on the observer context provided.

## When to Use

- When a user wants to build a saliency detection system that accounts for different user personas (e.g., "a foodie" vs. "a photographer" looking at the same scene)
- When implementing an iterative VLM-guided segmentation refinement loop where a language model evaluates and corrects mask quality
- When constructing annotation pipelines that generate diverse instruction-mask pairs from existing segmentation datasets using MLLMs
- When the user needs to segment objects in images based on natural language intent descriptions (e.g., "I want to check my email" applied to a desk scene)
- When building agentic computer vision pipelines that combine bounding-box prediction from a VLM with pixel-level segmentation from SAM2
- When creating datasets where the same image has multiple valid annotations depending on observer context

## Key Technique

Traditional salient object detection assumes one correct answer per image -- the single most "salient" region. OC-SOD reframes this as a subjective, observer-dependent task. The observer's profile (a preference like "nature enthusiast" or an intent like "find something to eat") is encoded as a natural language instruction and fed alongside the image to a multimodal LLM. This textual conditioning steers both what the model considers salient and how it refines its predictions.

The core innovation is **OC-SODAgent**, which implements a "Perceive-Reflect-Adjust" loop using two models in tandem: a Vision-Language Model (Qwen-VL family) for reasoning and a segmentation model (SAMv2-Hiera-Large) for mask generation. In the initial pass, the VLM reads the image and observer instruction, then outputs bounding boxes around candidate salient regions plus referring descriptions. SAM2 converts these boxes into pixel-level masks. Then, for up to K iterations (optimal K=3), the system renders the current mask contours back onto the image, asks the VLM to reflect on whether the segmentation is correct (checking for container inclusion errors, occlusion issues, semantic misalignment), and generates corrected bounding boxes. SAM2 re-segments, and the cycle repeats until the VLM signals "Finish" or the iteration budget is exhausted.

The data annotation pipeline is equally important: it uses MLLMs to categorize images (free-viewing vs. complex scenes requiring subjective interpretation), generate observer portraits and intent descriptions, and verify annotation quality. This produces instruction-mask pairs where each image can have 1-5 different valid saliency annotations depending on observer context. The OC-SODBench dataset built this way contains 33k images with 152k instruction-mask pairs across three modes: free-viewing, preference-driven, and intent-driven.

## Step-by-Step Workflow

1. **Define the observer context format.** Structure observer input as one of three modes: (a) free-viewing with a fixed prompt about visual contrast and semantic meaning, (b) preference-driven with an observer portrait string (e.g., "A wildlife photographer interested in animal behavior"), or (c) intent-driven with a task goal string (e.g., "I need to find the exit sign").

2. **Set up the VLM + SAM2 backbone.** Initialize a multimodal LLM (Qwen-VL-8B-Instruct or similar) as the reasoning engine and SAMv2-Hiera-Large as the segmentation engine. The VLM must accept image + text input and output structured bounding boxes. SAM2 must accept image + box prompts and output binary masks.

3. **Compose the initial prediction prompt.** Format the instruction as: `"Here is the observer's [portrait/intent]: {context}. Identify and segment the most salient regions according to the observer's [interest and preference / intent]."` Pass this with the image to the VLM to obtain initial bounding boxes (B_0) and referring descriptions (D).

4. **Generate initial masks with SAM2.** Feed the image and predicted bounding boxes into SAM2 to produce pixel-level segmentation masks (M_0). Each bounding box produces one mask region.

5. **Enter the Perceive-Reflect-Adjust loop (iterate up to K=3 times).**
   - **Perceive:** Extract contours from the current masks and render them as colored overlays on the original image (green boxes for predicted regions, red borders for segmentation boundaries).
   - **Reflect:** Send the annotated image, the referring descriptions, and a reflection prompt to the VLM. The reflection prompt asks the model to evaluate: Does the mask include the container instead of the object? Is there occlusion not accounted for? Does the segmented region semantically match the observer's intent? The VLM outputs corrected bounding boxes and a termination signal ("Continue" or "Finish").
   - **Adjust:** If not terminated, feed corrected boxes to SAM2 for re-segmentation. Update masks.

6. **Apply termination logic.** Stop iterating when the VLM outputs "Finish" or when K_max iterations are reached. Return the final mask set and metadata (iteration count, confidence).

7. **Post-process and evaluate.** Compute standard saliency metrics (S-measure for structural similarity, max F-measure, E-measure for alignment) and segmentation metrics (gIoU per instance, cIoU for dataset-level consistency) to validate output quality.

8. **For dataset construction: build the annotation pipeline.** Filter source images (remove targets < 0.1% of image area), categorize with VLM captioning (free-viewing vs. complex), generate observer portraits/intents with the MLLM, pair with existing ground-truth masks, and verify with a reasoning-capable VLM (e.g., Qwen-VL-Thinking) plus manual expert review.

## Concrete Examples

**Example 1: Implementing the Perceive-Reflect-Adjust loop in Python**

User: "Build me an iterative segmentation refinement agent that uses a VLM to evaluate and correct SAM2 masks based on a user intent description."

Approach:
1. Define the `OCSODAgent` class with `perceive()`, `reflect()`, and `adjust()` methods
2. Wire up the VLM client and SAM2 model
3. Implement the iterative loop with structured output parsing

Output:
```python
import torch
from sam2.sam2_image_predictor import SAM2ImagePredictor
from transformers import Qwen2_5_VLForConditionalGeneration, AutoProcessor

class OCSODAgent:
    def __init__(self, vlm_model_id, sam2_checkpoint, max_iterations=3):
        self.max_iterations = max_iterations
        self.vlm = Qwen2_5_VLForConditionalGeneration.from_pretrained(vlm_model_id)
        self.processor = AutoProcessor.from_pretrained(vlm_model_id)
        self.sam2 = SAM2ImagePredictor.from_pretrained(sam2_checkpoint)

    def initial_predict(self, image, instruction):
        """VLM generates bounding boxes and descriptions from image + observer instruction."""
        prompt = f"Given this image and instruction: '{instruction}', "
        prompt += "output bounding boxes [x1,y1,x2,y2] for salient regions and a referring description for each."
        boxes, descriptions = self._query_vlm(image, prompt)
        masks = self._segment_with_sam2(image, boxes)
        return masks, boxes, descriptions

    def perceive(self, image, masks, boxes):
        """Render current mask contours onto image for visual feedback."""
        annotated = image.copy()
        for mask in masks:
            contour = extract_contour(mask)
            annotated = draw_contour(annotated, contour, color="red")
        for box in boxes:
            annotated = draw_box(annotated, box, color="green")
        return annotated

    def reflect(self, annotated_image, descriptions, instruction):
        """VLM evaluates mask quality and suggests corrections."""
        reflection_prompt = (
            f"The green boxes show predicted regions. Red borders show segmentation results. "
            f"Descriptions: {descriptions}. Original instruction: '{instruction}'. "
            f"Evaluate: (1) Does the mask capture the object, not its container? "
            f"(2) Is occlusion handled? (3) Does segmentation match the intent? "
            f"Output corrected boxes or 'Finish' if satisfactory."
        )
        result = self._query_vlm(annotated_image, reflection_prompt)
        return result  # (corrected_boxes, terminate_flag)

    def adjust(self, image, corrected_boxes):
        """Re-segment with corrected bounding boxes."""
        return self._segment_with_sam2(image, corrected_boxes)

    def iterative_refine(self, image, instruction):
        """Full Perceive-Reflect-Adjust loop."""
        masks, boxes, descs = self.initial_predict(image, instruction)
        for k in range(self.max_iterations):
            annotated = self.perceive(image, masks, boxes)
            corrected_boxes, should_stop = self.reflect(annotated, descs, instruction)
            if should_stop:
                break
            masks = self.adjust(image, corrected_boxes)
            boxes = corrected_boxes
        return masks, {"iterations": k + 1, "final_boxes": boxes}
```

**Example 2: Generating observer-conditioned prompts for a dataset**

User: "I have a dataset of kitchen images with existing segmentation masks. Generate diverse observer-conditioned prompts so the same image gets different saliency annotations."

Approach:
1. Categorize each image as free-viewing or complex using VLM captioning
2. For complex images, generate 2-3 observer portraits and 1-2 intent descriptions
3. Pair each prompt with the appropriate subset of existing ground-truth masks

Output:
```python
PROMPT_TEMPLATES = {
    "free_viewing": (
        "Identify and segment the most salient regions according to "
        "visual context, color contrast, and semantic meaning."
    ),
    "preference": (
        "Here is the observer's portrait: {portrait}. "
        "Identify and segment the most salient regions according to "
        "the observer's interest and preference."
    ),
    "intent": (
        "Here is the observer's intent: {intent}. "
        "Identify and segment the most salient regions according to this intent."
    ),
}

def generate_annotations(image, existing_masks, vlm_client):
    caption = vlm_client.caption(image)
    category = vlm_client.classify(
        image, prompt="Is there a single unambiguous salient object (free-viewing) "
        "or multiple objects that different observers might prioritize (complex)?"
    )
    annotations = []
    if category == "free_viewing":
        annotations.append({
            "instruction": PROMPT_TEMPLATES["free_viewing"],
            "masks": [select_primary_mask(existing_masks)],
        })
    else:
        # Generate 2-3 observer portraits
        portraits = vlm_client.generate(
            image, "Generate 3 distinct observer portraits who would focus on "
            "different objects in this kitchen scene. Format: one per line."
        ).split("\n")
        for portrait in portraits:
            target_mask = vlm_client.select_mask(image, existing_masks, portrait)
            annotations.append({
                "instruction": PROMPT_TEMPLATES["preference"].format(portrait=portrait),
                "masks": [target_mask],
            })
        # Generate 1-2 intent descriptions
        intents = vlm_client.generate(
            image, "Generate 2 task intents someone might have in this kitchen scene."
        ).split("\n")
        for intent in intents:
            target_mask = vlm_client.select_mask(image, existing_masks, intent)
            annotations.append({
                "instruction": PROMPT_TEMPLATES["intent"].format(intent=intent),
                "masks": [target_mask],
            })
    return annotations

# Example output for a kitchen image:
# [
#   {"instruction": "Here is the observer's portrait: A home baker looking for baking tools. ...", "masks": [whisk_mask]},
#   {"instruction": "Here is the observer's portrait: A nutritionist evaluating meal prep. ...", "masks": [vegetables_mask]},
#   {"instruction": "Here is the observer's intent: I want to make coffee. ...", "masks": [coffee_machine_mask]},
# ]
```

**Example 3: Evaluating observer-centric saliency predictions**

User: "How do I evaluate my OC-SOD model's output against multiple valid ground truths?"

Approach:
1. Compute per-instance gIoU between predicted mask and the observer-specific ground truth
2. Aggregate with cIoU across the dataset for consistency
3. Report S-measure, F-measure, and E-measure for structural quality

Output:
```python
import numpy as np

def generalized_iou(pred_mask, gt_mask):
    intersection = np.logical_and(pred_mask, gt_mask).sum()
    union = np.logical_or(pred_mask, gt_mask).sum()
    pred_area = pred_mask.sum()
    gt_area = gt_mask.sum()
    enclosing_area = compute_enclosing_area(pred_mask, gt_mask)
    iou = intersection / (union + 1e-8)
    giou = iou - (enclosing_area - union) / (enclosing_area + 1e-8)
    return giou

def cumulative_iou(all_preds, all_gts):
    total_intersection = sum(np.logical_and(p, g).sum() for p, g in zip(all_preds, all_gts))
    total_union = sum(np.logical_or(p, g).sum() for p, g in zip(all_preds, all_gts))
    return total_intersection / (total_union + 1e-8)

def evaluate_oc_sod(predictions, ground_truths):
    """Each prediction is paired with its observer-specific ground truth."""
    giou_scores = [generalized_iou(p, g) for p, g in zip(predictions, ground_truths)]
    ciou = cumulative_iou(predictions, ground_truths)
    return {
        "mean_gIoU": np.mean(giou_scores),
        "cIoU": ciou,
        # Add S-measure, F-measure, E-measure from standard saliency eval libraries
    }
```

## Best Practices

- **Do:** Keep iteration count at K=3 for the Perceive-Reflect-Adjust loop. Ablations show diminishing returns beyond this, and inference cost scales linearly with iterations.
- **Do:** Render both bounding boxes (green) and mask contours (red) on the feedback image passed to the VLM during the Reflect step. The VLM needs both spatial reference and segmentation detail to make accurate corrections.
- **Do:** Use structured output parsing for VLM bounding box predictions. Enforce a consistent format (e.g., JSON with `[x1, y1, x2, y2]` arrays) to avoid parsing failures that break the loop.
- **Do:** Separate observer context into distinct modes (free-viewing, preference, intent) rather than mixing them. Each mode has different prompt structures and evaluation characteristics.
- **Avoid:** Using a single ground-truth mask when evaluating observer-centric models. Each prediction must be evaluated against the ground truth corresponding to its specific observer instruction.
- **Avoid:** Skipping the data filtering step when building annotation pipelines. Objects smaller than 0.1% of image area or semantically uninformative regions produce noisy training signal.

## Error Handling

- **VLM outputs malformed bounding boxes:** Validate all box coordinates are within image bounds and have positive area. If invalid, fall back to the previous iteration's boxes rather than crashing.
- **SAM2 produces empty masks:** This happens when bounding boxes are too small or in featureless regions. Apply a minimum box size threshold (e.g., 16x16 pixels) and retry with slightly expanded boxes.
- **Reflection loop never terminates:** Always enforce K_max as a hard ceiling. If the VLM oscillates between different box sets across iterations, take the mask from the iteration with the best self-reported confidence.
- **Observer instruction is ambiguous:** When the VLM generates descriptions that don't clearly map to visible objects, add a verification step that checks whether the referring description matches any region in the image above a confidence threshold.

## Limitations

- Inference cost is substantial: each iteration requires a full VLM forward pass plus SAM2 segmentation. At K=3, this means 4 VLM calls and 4 SAM2 calls per image.
- The quality ceiling is bounded by the VLM's visual understanding. If the base VLM cannot distinguish between semantically similar objects (e.g., two similar cups), the refinement loop cannot fix this.
- Observer portraits and intents must be expressible in natural language. Implicit or subconscious visual preferences that a human cannot articulate are not captured.
- The approach assumes access to a capable multimodal LLM (8B+ parameters). Smaller models degrade performance significantly -- ablations show gIoU dropping from 47.07 (32B) to 36.97 (8B).
- Real-time applications are impractical with the iterative loop. Single-pass distilled models may be needed for latency-sensitive deployments.

## Reference

**Paper:** [Revisiting Salient Object Detection from an Observer-Centric Perspective](https://arxiv.org/abs/2602.06369v1) (Zhang et al., 2026). Look for Section 4 (OC-SODAgent architecture and the Perceive-Reflect-Adjust formalization) and Section 3 (the five-step data annotation pipeline using MLLMs).

**Code:** [https://github.com/Dustzx/OC_SOD](https://github.com/Dustzx/OC_SOD) -- reference implementation with SAM2 + Qwen-VL integration.