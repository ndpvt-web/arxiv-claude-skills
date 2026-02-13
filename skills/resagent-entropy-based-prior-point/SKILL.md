---
name: "resagent-entropy-based-prior-point"
description: "Implement entropy-guided coarse-to-fine visual grounding pipelines for referring expression segmentation and point-prompt selection. Use when: 'build a referring expression segmentation system', 'select informative point prompts from a bounding box', 'entropy-based candidate point discovery', 'validate point prompts with visual reasoning', 'coarse-to-fine segmentation with SAM', 'visually verify object locations instead of text coordinate reasoning'."
---

# ResAgent: Entropy-Based Prior Point Discovery and Visual Reasoning

This skill enables Claude to build coarse-to-fine visual grounding and segmentation systems that use **entropy-based spatial sampling** to select maximally informative point prompts from bounding boxes, and **visual-semantic reasoning** (rendering markers on images and asking VQA questions) to validate those points — rather than relying on unreliable textual coordinate reasoning. The approach, from the ResAgent paper, consistently outperforms naive center-point or grid-sampling strategies when feeding point prompts to segmentation models like SAM.

## When to Use

- When the user needs to build a **referring expression segmentation** (RES) pipeline that takes a text description and an image, then produces a pixel-level mask of the described object
- When the user has **bounding box detections** and needs to select high-quality positive/negative point prompts for SAM or similar prompt-based segmentation models
- When the user wants to **replace naive center-point or uniform grid sampling** with an information-theoretic point selection strategy
- When the user is building a system that must **distinguish a target from visually similar distractors** in the same bounding box region
- When the user wants to **validate spatial predictions visually** (by rendering markers on images and querying a VLM) instead of parsing text-based coordinate outputs
- When implementing any **agentic vision pipeline** with a coarse-to-fine refinement loop: detect → sample → verify → segment

## Key Technique

### Problem: Naive Point Prompts Fail

Standard MLLM-based segmentation pipelines predict a bounding box, then feed its center (or a grid of points) to SAM. This fails because: (1) the center of a coarse box may land on background or an occluding object, and (2) uniform grids waste prompt budget on redundant low-information regions. ResAgent reframes point selection as an **information maximization** problem.

### Entropy-Based Point Discovery (EBD)

Each pixel inside a predicted bounding box is modeled as a Bernoulli random variable — it either belongs to the target object or it doesn't. The probability of membership is estimated via two geometric signals: distance from the box center `dc` (closer = more likely positive) and distance from the nearest edge `de` (closer to edge = more likely negative). These are combined through a logistic function:

```
p(pt) = sigmoid(a - b * dc_norm(pt) + c * de_norm(pt))
```

Shannon entropy `H(pt) = -p*log(p) - (1-p)*log(1-p)` peaks at `p ≈ 0.5`, which identifies **boundary-proximal positives** and **center-proximal negatives** — exactly the most informative points for disambiguating object extent. A dual-queue greedy search (inward from edges + outward from center) harvests the top-K highest-entropy candidates.

### Vision-Based Reasoning (VBR)

Instead of asking an MLLM to reason about coordinates as text tokens (which is unreliable), each candidate point is validated **visually**: a colored marker (star, circle) is rendered on the image at that location, and the VLM is asked a binary question: *"Is the red-colored star on the object referred to by '[expression]'?"*. Token-level probabilities for "yes"/"no" variants are aggregated to produce a confidence score. Points exceeding a threshold η are classified as positive or negative prompts. Early stopping triggers once 2 positive and 1 negative point are collected.

## Step-by-Step Workflow

1. **Accept inputs**: Take an image and a free-form text expression describing the target object (e.g., "the dog on the left wearing a red collar").

2. **Predict a coarse bounding box**: Query an MLLM (or any object detector) with the expression to obtain a bounding box `[x_min, y_min, x_max, y_max]` in absolute pixel coordinates. Normalize coordinates to the image dimensions.

3. **Build the entropy field over the bounding box**: For each candidate point inside the box, compute normalized center-distance `dc_norm ∈ [0,1]` and edge-distance `de_norm ∈ [0,1]`. Apply the logistic membership model `p(pt) = sigmoid(a - b*dc_norm + c*de_norm)` with tuned hyperparameters (defaults: `a=0, b=3, c=2`). Compute Shannon entropy `H(pt)` at each location.

4. **Select top-K candidate points via dual-queue greedy search**: Initialize an internal queue (scanning inward from each box edge along normals) and an external queue (expanding outward from center along radial rays). Rank all harvested candidates by entropy `H(pt)` descending. Retain the top-K points (typically K=6–10).

5. **Render visual markers on the image**: For each candidate point, overlay a distinctly colored marker (e.g., red star, blue circle, green triangle) at that pixel location. Use non-overlapping colors and record the color-marker mapping.

6. **Query the VLM for binary point validation**: For each marked point, send the annotated image to the VLM with: `"Answer strictly yes or no: Is the {color}-colored {marker} on the object referred to by '{expression}' in the picture?"`. Aggregate top-5 token logits for yes/no variants to compute confidence `P(positive)`.

7. **Classify points with confidence thresholding**: Points with `P(positive) > η` (default η=0.6) are positive prompts; points with `P(positive) < (1-η)` are negative prompts. Discard ambiguous points.

8. **Apply early stopping**: Stop validation once you have at least 2 positive points and 1 negative point. This minimizes VLM query cost.

9. **Decode the segmentation mask**: Feed the original image, bounding box, positive points, and negative points to SAM: `mask = SAM(image, bbox, positive_pts, negative_pts)`. Select the highest-confidence mask from SAM's multi-mask output.

10. **Post-process and return**: Apply optional CRF refinement or morphological cleanup. Return the binary mask, the validated point set, and confidence scores.

## Concrete Examples

**Example 1: Building an entropy-guided point sampler for SAM**

User: "I have bounding boxes from a detector and I need better point prompts for SAM instead of just using the center point."

Approach:
1. Implement the entropy field computation over each bounding box
2. Use dual-queue sampling to find high-entropy candidates
3. Return ranked point prompts

Output (Python implementation):
```python
import numpy as np
from scipy.special import expit as sigmoid

def entropy_point_discovery(bbox, num_candidates=8, a=0.0, b=3.0, c=2.0):
    """Select maximally informative point prompts from a bounding box.

    Args:
        bbox: [x_min, y_min, x_max, y_max] in pixel coordinates
        num_candidates: number of top-entropy points to return
        a, b, c: logistic model hyperparameters

    Returns:
        List of (x, y, entropy) tuples sorted by entropy descending
    """
    x_min, y_min, x_max, y_max = bbox
    w, h = x_max - x_min, y_max - y_min
    cx, cy = (x_min + x_max) / 2, (y_min + y_max) / 2

    # Sample candidate grid (denser than final output)
    xs = np.linspace(x_min + 1, x_max - 1, max(int(w / 4), 10))
    ys = np.linspace(y_min + 1, y_max - 1, max(int(h / 4), 10))
    grid_x, grid_y = np.meshgrid(xs, ys)
    pts = np.stack([grid_x.ravel(), grid_y.ravel()], axis=1)

    # Normalized center distance
    dc = np.sqrt(((pts[:, 0] - cx) / (w / 2)) ** 2 +
                 ((pts[:, 1] - cy) / (h / 2)) ** 2)
    dc_norm = np.clip(dc / dc.max(), 0, 1)

    # Normalized edge distance (min distance to any edge)
    de = np.minimum(
        np.minimum(pts[:, 0] - x_min, x_max - pts[:, 0]),
        np.minimum(pts[:, 1] - y_min, y_max - pts[:, 1])
    )
    de_norm = np.clip(de / de.max(), 0, 1)

    # Membership probability and entropy
    p = sigmoid(a - b * dc_norm + c * de_norm)
    entropy = -p * np.log(p + 1e-10) - (1 - p) * np.log(1 - p + 1e-10)

    # Select top-K by entropy
    top_k_idx = np.argsort(entropy)[-num_candidates:][::-1]
    return [(int(pts[i, 0]), int(pts[i, 1]), float(entropy[i]))
            for i in top_k_idx]
```

**Example 2: Full coarse-to-fine RES pipeline with visual validation**

User: "Build a referring expression segmentation pipeline that uses a VLM for grounding and SAM for masks."

Approach:
1. Use the VLM to predict a bounding box from the expression
2. Run entropy-based point discovery inside that box
3. Validate each candidate by rendering markers and querying the VLM
4. Feed validated points + box to SAM

Output (pipeline sketch):
```python
from PIL import Image, ImageDraw
import json

MARKER_COLORS = ["red", "blue", "green", "yellow", "cyan", "magenta"]

def visual_point_validation(image, candidates, expression, vlm_client, eta=0.6):
    """Validate candidate points via visual marker rendering + VQA."""
    positive_pts, negative_pts = [], []

    for i, (x, y, _entropy) in enumerate(candidates):
        color = MARKER_COLORS[i % len(MARKER_COLORS)]

        # Render marker on image copy
        img_marked = image.copy()
        draw = ImageDraw.Draw(img_marked)
        r = max(image.width, image.height) // 80
        draw.ellipse([x - r, y - r, x + r, y + r], fill=color, outline="white")

        # Query VLM with binary question
        prompt = (
            f"Answer strictly yes or no: Is the {color}-colored circle "
            f"on the object referred to by '{expression}' in the picture?"
        )
        response = vlm_client.query(img_marked, prompt, return_logprobs=True)
        p_yes = aggregate_yes_probability(response.top_logprobs)

        if p_yes > eta:
            positive_pts.append((x, y))
        elif p_yes < (1 - eta):
            negative_pts.append((x, y))

        # Early stopping
        if len(positive_pts) >= 2 and len(negative_pts) >= 1:
            break

    return positive_pts, negative_pts


def resagent_pipeline(image, expression, vlm_client, sam_model):
    """Full coarse-to-fine referring expression segmentation."""
    # Stage 1: Bounding box from VLM
    bbox_prompt = (
        f"Identify the object referred to by '{expression}' and return "
        f"the bounding box as JSON: {{\"bbox\": [x_min, y_min, x_max, y_max]}}"
    )
    bbox_response = vlm_client.query(image, bbox_prompt)
    bbox = json.loads(bbox_response)["bbox"]

    # Stage 2: Entropy-guided point discovery
    candidates = entropy_point_discovery(bbox, num_candidates=8)

    # Stage 3: Vision-based validation
    pos_pts, neg_pts = visual_point_validation(
        image, candidates, expression, vlm_client
    )

    # Stage 4: SAM mask decoding
    mask = sam_model.predict(
        image=image,
        box=bbox,
        point_coords=pos_pts + neg_pts,
        point_labels=[1] * len(pos_pts) + [0] * len(neg_pts)
    )
    return mask, bbox, pos_pts, neg_pts
```

**Example 3: Adapting entropy sampling for interactive annotation tools**

User: "I'm building an annotation tool where users draw rough boxes and I need to suggest click points."

Approach:
1. Take the user's rough bounding box as input
2. Compute the entropy field and return the top-3 most informative points to suggest
3. Visualize entropy as a heatmap overlay so the user can see why those points matter

Output:
```python
import matplotlib.pyplot as plt
import numpy as np

def visualize_entropy_field(image, bbox, a=0.0, b=3.0, c=2.0):
    """Render entropy heatmap over bounding box region."""
    x_min, y_min, x_max, y_max = bbox
    w, h = x_max - x_min, y_max - y_min
    cx, cy = (x_min + x_max) / 2, (y_min + y_max) / 2

    ys, xs = np.mgrid[y_min:y_max, x_min:x_max]
    dc = np.sqrt(((xs - cx) / (w / 2)) ** 2 + ((ys - cy) / (h / 2)) ** 2)
    dc_norm = dc / (dc.max() + 1e-8)
    de = np.minimum(
        np.minimum(xs - x_min, x_max - xs),
        np.minimum(ys - y_min, y_max - ys)
    )
    de_norm = de / (de.max() + 1e-8)

    from scipy.special import expit as sigmoid
    p = sigmoid(a - b * dc_norm + c * de_norm)
    entropy = -p * np.log(p + 1e-10) - (1 - p) * np.log(1 - p + 1e-10)

    fig, ax = plt.subplots(1, 1, figsize=(8, 6))
    ax.imshow(image)
    ax.imshow(entropy, extent=[x_min, x_max, y_max, y_min],
              alpha=0.5, cmap="hot")
    ax.set_title("Entropy Field — bright = most informative for point prompts")
    plt.savefig("entropy_heatmap.png", dpi=150, bbox_inches="tight")
    return entropy
```

## Best Practices

- **Do:** Tune the logistic hyperparameters `(a, b, c)` to your domain. The defaults `(0, 3, 2)` work well for natural images but tighter boxes (from strong detectors) may benefit from lower `b` to spread entropy further inward.
- **Do:** Use distinct, high-contrast marker colors when rendering visual prompts for VLM validation. Avoid colors that blend with common image content (e.g., green on vegetation).
- **Do:** Aggregate token-level logits for "yes"/"Yes"/"YES" and "no"/"No"/"NO" variants to avoid probability splitting across surface forms.
- **Do:** Apply early stopping (2 positive + 1 negative) to minimize VLM query cost. More points rarely improve SAM output but always increase latency.
- **Avoid:** Using text-based coordinate reasoning (e.g., "Is the point at (142, 305) on the dog?"). VLMs are unreliable at mapping text tokens to spatial positions — visual marker rendering is consistently >40% more accurate.
- **Avoid:** Sampling points uniformly or only at the bounding box center. The entropy-based approach specifically targets the boundary region where SAM most needs disambiguation signal.

## Error Handling

- **Bounding box misses the target entirely**: If the VLM returns a clearly wrong box (e.g., empty region), fall back to a wider search area or re-query with a rephrased expression. Check that predicted boxes have reasonable area relative to the image.
- **No positive points pass validation**: Lower the confidence threshold η or increase the candidate pool size K. If still failing, the bounding box is likely wrong — escalate to re-detection.
- **VLM gives ambiguous answers**: When `P(yes)` clusters near 0.5 for all candidates, the expression may be ambiguous or the target too small. Flag this to the user rather than forcing a low-confidence prediction.
- **SAM produces multiple disjoint mask components**: This often indicates conflicting positive/negative points. Filter the smallest components by area or re-validate the points with stricter threshold.

## Limitations

- Requires access to both a VLM (for box prediction and visual validation) and SAM (for mask decoding) — not suitable for lightweight or single-model deployments.
- The entropy field is a geometric prior based on box geometry, not learned from data. It assumes objects are roughly centered in their bounding boxes, which breaks for highly elongated, off-center, or multi-part objects.
- Visual marker rendering adds latency per candidate point. For real-time applications, consider pre-computing the entropy field and skipping VLM validation for high-confidence center points.
- The technique targets single-object segmentation per expression. Multi-object expressions ("all the dogs") require upstream decomposition.
- Performance depends heavily on bounding box quality. A box that cuts off part of the target will bias the entropy field and produce poor point candidates.

## Reference

**Paper:** [ResAgent: Entropy-based Prior Point Discovery and Visual Reasoning for Referring Expression Segmentation](https://arxiv.org/abs/2601.16394v1) — Look for Section 3 (Method) detailing the EBD entropy formulation and dual-queue search, and Section 3.3 on VBR visual marker validation with token probability aggregation.