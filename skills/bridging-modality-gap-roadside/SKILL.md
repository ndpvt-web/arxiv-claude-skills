---
name: "bridging-modality-gap-roadside"
description: "Build training-free pipelines that convert sparse 3D LiDAR point clouds into depth-encoded 2D images for classification by Vision-Language Models (CLIP, etc.). Covers the full workflow: point cloud denoising, temporal frame fusion, canonical orientation, orthographic projection, morphological cleanup, bilateral smoothing, and few-shot VLM prompting with semantic anchoring. Use when: 'classify vehicles from roadside LiDAR', 'convert point clouds to images for VLM', 'few-shot 3D object classification without training', 'bridge LiDAR to vision-language model', 'bootstrap labeled dataset from unlabeled LiDAR', 'cold start vehicle classifier from point clouds'."
---

# Bridging the Modality Gap: Training-Free VLM Classification of LiDAR Point Clouds

This skill enables Claude to design and implement pipelines that transform sparse 3D LiDAR point cloud data into clean, depth-encoded 2D images suitable for classification by off-the-shelf Vision-Language Models (e.g., CLIP ViT-L/14) — entirely without model fine-tuning. The core technique, from Li et al. (2026), achieves competitive F1 scores (~0.62) on 20-class vehicle classification using only 16–30 examples per class, and can bootstrap fully supervised models (PointNet, F1 ~0.71) via a Cold Start strategy.

## When to Use

- When the user wants to classify objects (vehicles, infrastructure, etc.) from roadside or traffic LiDAR scans without training a custom deep learning model
- When the user needs to convert sparse 3D point clouds into 2D image representations for input to vision or vision-language models
- When building a few-shot classification system for 3D sensor data using CLIP or similar VLMs
- When the user wants to bootstrap a labeled training dataset from unlabeled LiDAR data (Cold Start)
- When the user asks about bridging the modality gap between point clouds and image-based foundation models
- When implementing a denoising-to-projection pipeline for LiDAR frame sequences (noise removal, registration, rectification, rendering)

## Key Technique

**The Modality Bridge.** Vision-Language Models like CLIP excel at few-shot image classification but cannot ingest raw 3D point clouds. This framework bridges that gap with a six-stage deterministic pipeline: (1) statistical outlier removal on each LiDAR frame, (2) temporal fusion of multiple frames via probabilistic registration (FilterReg), (3) canonical orientation alignment so every vehicle faces the same direction, (4) orthographic projection onto the YZ plane with minimum-depth encoding, (5) morphological opening to remove projection artifacts, and (6) bilateral filtering for edge-preserving smoothing. The output is a single-channel grayscale image where pixel intensity encodes lateral depth — visually resembling a side-view silhouette with depth shading.

**Few-Shot VLM Prompting with Semantic Anchoring.** The generated depth images are fed to CLIP (ViT-L/14 recommended) using few-shot in-context learning. For each class, k support examples (k=1..30) are encoded into visual prototypes. An optional text embedding (class description) can be fused with visual prototypes: `p_fused = (1-w) * p_visual + w * e_text`. A critical finding is the **Semantic Anchor effect**: at ultra-low shot counts (k < 4), text guidance at weight w=0.2 regularizes predictions and prevents collapse; but beyond k=8, text embeddings degrade accuracy due to semantic mismatch between natural-language vehicle descriptions and depth-image appearance. Set w=0 for k >= 8.

**Cold Start Bootstrapping.** The training-free VLM pipeline generates pseudo-labels for unlabeled LiDAR data. These labels (even at ~62% accuracy) are sufficient to train a lightweight supervised 3D classifier (e.g., PointNet) that achieves F1=0.71 with 4ms inference — suitable for real-time deployment. This eliminates the expensive manual annotation phase entirely.

## Step-by-Step Workflow

1. **Ingest and parse raw LiDAR frames.** Load point cloud data (`.pcd`, `.ply`, `.bin`, or `.npy` format) for each tracked object. Each frame is a set of (x, y, z) coordinates from sensors like Velodyne VLP-32c. Group frames by object/vehicle track ID.

2. **Apply statistical outlier removal (SOR) per frame.** For each frame: voxel-downsample at 0.05m resolution, compute mean distance μ_i to k-nearest neighbors for each point, then remove points where μ_i > μ_global + α·σ_global (α=1.0). Use Open3D's `statistical_outlier_removal` or implement manually.

3. **Fuse temporal frames via probabilistic registration.** Align and merge all filtered frames for the same tracked vehicle into a single dense point cloud. Use FilterReg (Gao & Tedrake, 2019) or ICP as a fallback. This compensates for the sparsity of individual LiDAR sweeps by accumulating geometry across time.

4. **Rectify orientation to a canonical coordinate system.** Compute the ground-plane normal (Z-axis), negate the normalized motion vector for the Y-axis (longitudinal), and derive X-axis via cross product. Apply the rotation matrix so every vehicle is oriented consistently — front facing the same direction, side profile visible from the YZ plane.

5. **Project orthographically onto the YZ plane with minimum-depth encoding.** Create a 2D image grid over the YZ extent of the point cloud. For each pixel (y, z), assign intensity = min(x) across all points within spatial tolerance δ of that grid cell. This produces a single-channel grayscale depth image — a side-view silhouette where brightness encodes lateral distance.

6. **Apply morphological opening.** Perform erosion then dilation using an elliptical structuring element of size s×s (tune s based on image resolution, typically 3–5 pixels). This removes thin projection artifacts and isolated noise pixels while preserving vehicle shape. Use OpenCV's `cv2.morphologyEx(img, cv2.MORPH_OPEN, kernel)`.

7. **Apply bilateral filtering for edge-preserving smoothing.** Smooth the image using bilateral filtering: `cv2.bilateralFilter(img, d, sigma_color, sigma_space)`. Set sigma_space to control spatial smoothing extent and sigma_color to preserve depth edges (typical starting points: sigma_space=10, sigma_color=50). This produces the final clean depth proxy image.

8. **Encode images and classify with CLIP.** Load a pretrained CLIP model (ViT-L/14 recommended). Encode k support images per class into visual prototype embeddings (mean pooling). For k < 4, fuse with text embeddings at w=0.2: `p_fused = 0.8 * p_visual + 0.2 * encode_text(class_description)`. For k >= 8, use pure visual prototypes (w=0). Classify query images by nearest prototype in embedding space (cosine similarity).

9. **Evaluate with F1-score over multiple runs.** Run classification 10 times with different random support-set draws. Report mean F1 and standard deviation across runs. Inspect confusion matrices for systematic misclassifications between visually similar classes.

10. **Optionally bootstrap a supervised model (Cold Start).** Use VLM-generated labels as pseudo-ground-truth to train a lightweight 3D classifier (PointNet, DGCNN, or a simple ViT fine-tune) on the original point clouds. This yields faster inference (~4ms vs. VLM latency) and higher accuracy (F1 ~0.71) while requiring zero manual annotation.

## Concrete Examples

**Example 1: Classify roadside LiDAR vehicle tracks with CLIP**

User: "I have Velodyne LiDAR point cloud sequences for vehicles passing a highway checkpoint. I want to classify them into truck types (container sizes, tankers, platforms, etc.) without training a model."

Approach:
1. Load `.pcd` frame sequences grouped by vehicle track ID using Open3D
2. For each track, run SOR (voxel=0.05m, α=1.0), then fuse frames with ICP/FilterReg
3. Compute canonical orientation from motion vector and ground plane
4. Project each fused cloud to a 128×128 grayscale depth image (YZ plane, min-depth)
5. Clean with morphological opening (elliptical kernel 3×3) and bilateral filter (σ_sp=10, σ_c=50)
6. Load CLIP ViT-L/14, encode 16 support images per class
7. Classify queries by cosine similarity to mean visual prototypes

Output:
```python
import open3d as o3d
import numpy as np
import cv2
import clip
import torch

# Stage 1-2: Denoise a single frame
def denoise_frame(pcd, voxel_size=0.05, alpha=1.0):
    pcd_down = pcd.voxel_down_sample(voxel_size)
    cl, ind = pcd_down.remove_statistical_outlier(nb_neighbors=20, std_ratio=alpha)
    return pcd_down.select_by_index(ind)

# Stage 5: Orthographic depth projection
def project_to_depth_image(points, resolution=128, tolerance=0.05):
    y, z = points[:, 1], points[:, 2]
    x = points[:, 0]
    y_bins = np.linspace(y.min(), y.max(), resolution)
    z_bins = np.linspace(z.min(), z.max(), resolution)
    img = np.full((resolution, resolution), np.nan)
    for i in range(resolution - 1):
        for j in range(resolution - 1):
            mask = (y >= y_bins[i]) & (y < y_bins[i+1]) & (z >= z_bins[j]) & (z < z_bins[j+1])
            if mask.any():
                img[resolution - 1 - j, i] = x[mask].min()
    img_norm = np.nan_to_num(img, nan=0)
    if img_norm.max() > 0:
        img_norm = (255 * (img_norm - img_norm[img_norm > 0].min()) /
                    (img_norm.max() - img_norm[img_norm > 0].min())).clip(0, 255)
    return img_norm.astype(np.uint8)

# Stage 6-7: Morphological + bilateral cleanup
def clean_depth_image(img, kernel_size=3, sigma_sp=10, sigma_c=50):
    kernel = cv2.getStructuringElement(cv2.MORPH_ELLIPSE, (kernel_size, kernel_size))
    opened = cv2.morphologyEx(img, cv2.MORPH_OPEN, kernel)
    smoothed = cv2.bilateralFilter(opened, d=9, sigmaColor=sigma_c, sigmaSpace=sigma_sp)
    return smoothed

# Stage 8: CLIP few-shot classification
def classify_with_clip(query_img, support_sets, model, preprocess, device, w=0.0, text_embeds=None):
    query_tensor = preprocess(query_img).unsqueeze(0).to(device)
    with torch.no_grad():
        query_emb = model.encode_image(query_tensor)
        query_emb = query_emb / query_emb.norm(dim=-1, keepdim=True)
    best_sim, best_class = -1, None
    for cls_name, prototype in support_sets.items():
        p = prototype
        if w > 0 and text_embeds and cls_name in text_embeds:
            p = (1 - w) * prototype + w * text_embeds[cls_name]
            p = p / p.norm(dim=-1, keepdim=True)
        sim = (query_emb @ p.T).item()
        if sim > best_sim:
            best_sim, best_class = sim, cls_name
    return best_class, best_sim
```

**Example 2: Cold Start — bootstrap a PointNet model from VLM labels**

User: "I classified 500 vehicles using the CLIP pipeline above. Now I want to train a fast model for real-time deployment."

Approach:
1. Export VLM predictions as pseudo-labels paired with original point clouds
2. Split pseudo-labeled data 80/20 for train/validation
3. Train a PointNet classifier on raw (x, y, z) point clouds with pseudo-labels
4. Evaluate on a small manually-verified holdout set

Output:
```python
# Pseudo-label export format
pseudo_labels = {
    "track_001": {"points": np.array(...), "label": "40ft Container", "confidence": 0.87},
    "track_002": {"points": np.array(...), "label": "Bobtail", "confidence": 0.73},
    # ...
}

# Filter high-confidence labels for cleaner training data
train_data = {k: v for k, v in pseudo_labels.items() if v["confidence"] > 0.6}

# Train PointNet (using standard PointNet implementation)
# Expected result: F1 ~0.70, inference ~4ms per vehicle
```

**Example 3: Tuning the Semantic Anchor weight**

User: "I only have 2 examples per class. How should I set up the text fusion?"

Approach:
1. With k=2 (ultra-low-shot), enable text anchoring at w=0.2
2. Write descriptive class definitions (e.g., "A 53ft container is a semi-trailer carrying a 53-foot intermodal shipping container, rectangular profile with corrugated sides")
3. Encode text descriptions with CLIP's text encoder
4. Fuse: `p_fused = 0.8 * visual_prototype + 0.2 * text_embedding`

Output:
```python
# Text descriptions for semantic anchoring (k < 4)
class_descriptions = {
    "53ft Container": "A semi-trailer hauling a 53-foot intermodal shipping container with corrugated rectangular profile",
    "Bobtail": "A truck tractor driving without any attached trailer, showing exposed fifth wheel coupling",
    "Tank (Semi)": "A semi-trailer with a cylindrical tanker body for transporting liquids or gases",
    # ... one description per class
}

# Encode text
text_tokens = clip.tokenize(list(class_descriptions.values())).to(device)
with torch.no_grad():
    text_embeds = model.encode_text(text_tokens)
    text_embeds = text_embeds / text_embeds.norm(dim=-1, keepdim=True)

# Use w=0.2 for k < 4, w=0 for k >= 8
w = 0.2 if k < 4 else 0.0
```

## Best Practices

- **Do:** Use ViT-L/14 over ViT-B/32 for CLIP — the larger model yields measurably better F1 on depth-encoded images
- **Do:** Project onto the YZ plane (side view) rather than top-down — side profiles carry the most discriminative geometry for vehicle classification
- **Do:** Fuse multiple temporal frames before projection — single LiDAR sweeps are too sparse for reliable depth images, especially at range
- **Do:** Set text anchor weight w=0 once you have k >= 8 support examples — text embeddings cause semantic mismatch that hurts accuracy beyond ultra-low-shot regimes
- **Avoid:** Using perspective projection — orthographic projection preserves true proportions and eliminates distance-dependent distortion critical for size-based classification (e.g., 20ft vs. 53ft containers)
- **Avoid:** Skipping morphological opening — projection artifacts (thin spurs, isolated pixels) confuse CLIP's visual encoder and degrade classification
- **Avoid:** Fine-tuning the VLM — the entire value of this approach is zero parameter updates; fine-tuning on small depth-image sets leads to overfitting and removes generalization

## Error Handling

| Problem | Cause | Fix |
|---------|-------|-----|
| All-black depth image | Point cloud extent doesn't match image grid bounds | Compute YZ bounding box from actual point extents; add 10% padding |
| Noisy silhouettes with scattered dots | Insufficient SOR filtering | Decrease α from 1.0 to 0.5 or increase k-nearest-neighbors count |
| All vehicles classified as same class | Too few support examples or bad projection | Verify depth images visually; increase k to at least 16; check orientation rectification |
| F1 drops when adding text (k > 4) | Semantic Anchor degradation | Set w=0 for k >= 4; text descriptions in natural language don't match depth-image visual features |
| Temporal registration diverges | Large frame-to-frame motion or occlusion | Use motion-compensated ICP with initial transform from tracker; discard frames with < 50 points |
| Classes with similar profiles confused | Ambiguous side-view geometry (e.g., enclosed van SU vs. semi) | Add a second view (top-down XY projection) as a separate CLIP input channel; ensemble two views |

## Limitations

- **Single viewpoint only.** Roadside LiDAR captures one side of passing vehicles — rear/front geometry is unavailable, limiting discrimination between classes that differ only in those views
- **Sparse at range.** Vehicles far from the sensor produce very few points per frame even after temporal fusion, degrading image quality below usable thresholds at ~80m+
- **Semantic mismatch at higher k.** Natural-language class descriptions don't align with depth-image embeddings in CLIP's latent space, making text fusion counterproductive beyond ultra-low-shot
- **Not real-time.** The full pipeline (frame fusion + registration + projection + CLIP inference) runs slower than dedicated 3D models; use Cold Start bootstrapping for deployment
- **20-class taxonomy specific.** Validated on US highway truck classes; applying to different domains (construction equipment, rail cars, drones) requires new class definitions and support sets
- **Occlusion sensitivity.** Vehicles partially occluded by barriers or other vehicles produce incomplete silhouettes that may be misclassified

## Reference

Li, Y., Shang, B., & Wei, J. (2026). *Bridging the Modality Gap in Roadside LiDAR: A Training-Free Vision-Language Model Framework for Vehicle Classification*. arXiv:2602.09425v1. [https://arxiv.org/abs/2602.09425v1](https://arxiv.org/abs/2602.09425v1)

Key sections to consult: Algorithm 1 (full pipeline pseudocode), Table I (20-class taxonomy with descriptions), Table II (F1 comparison across shot counts and models), Figure 5 (Semantic Anchor weight ablation), Section IV-D (Cold Start strategy).