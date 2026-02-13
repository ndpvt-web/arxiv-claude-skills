---
name: "xai-clip-roi-guided-perturbation-framework"
description: "Build ROI-guided perturbation pipelines for explainable medical image segmentation using CLIP embeddings. Generates boundary-aware saliency maps by restricting perturbations to anatomically meaningful regions identified via vision-language models. Use when: 'explain medical segmentation predictions', 'build XAI pipeline for CT/MRI', 'generate saliency maps for organ segmentation', 'reduce perturbation cost for explainability', 'CLIP-guided occlusion sensitivity', 'ROI-aware LIME/RISE for medical images'."
---

# XAI-CLIP: ROI-Guided Perturbation Framework for Explainable Medical Image Segmentation

This skill enables Claude to implement ROI-guided perturbation-based explainability pipelines for medical image segmentation. The core technique uses a vision-language model (MediCLIP) to localize clinically relevant anatomical regions, then restricts perturbation-based explanation methods (Occlusion Sensitivity, LIME, RISE) to only those regions. This eliminates 75-95% of redundant perturbation evaluations on background tissue, producing cleaner saliency maps with up to 60% runtime reduction and 96.7% IoU improvement over brute-force perturbation.

## When to Use

- When the user asks to build an explainability pipeline for a medical image segmentation model (CT, MRI, ultrasound)
- When the user wants to generate saliency or attribution maps that highlight why a segmentation model predicted specific organ boundaries
- When the user needs to reduce the computational cost of perturbation-based XAI methods (occlusion sensitivity, LIME, RISE) on high-resolution medical volumes
- When the user asks to integrate CLIP or vision-language embeddings into an explanation workflow for medical imaging
- When the user wants anatomically coherent attribution maps instead of noisy, diffuse heatmaps that bleed into background regions
- When the user needs to validate or audit a deployed segmentation model (MedSAM, nnU-Net, U-Net) for clinical trust

## Key Technique

**The Problem:** Standard perturbation-based explainability methods (e.g., sliding-window occlusion) evaluate every patch in the image, including vast background regions with no clinical relevance. For a 512x512 medical image with 64x64 patches at stride 32, this means ~10,000 forward passes through the segmentation model. The resulting saliency maps are noisy, include spurious background attributions, and are too slow for clinical deployment.

**The XAI-CLIP Solution:** Before running any perturbation, use a vision-language model (MediCLIP) to produce a binary ROI mask identifying anatomically meaningful regions. MediCLIP encodes text prompts like "Liver", "Right Kidney", "Spleen" alongside the image into a shared CLIP embedding space, computes per-pixel similarity, and binarizes the result (threshold >= 0.5) into an ROI mask. Perturbations are then applied only to patches that overlap this ROI mask, reducing evaluated patches from ~10,000 to 500-2,000. Pixels outside the ROI are held constant (multiplier 1.0), ensuring background never contaminates the attribution signal.

**Why It Works:** The vision-language model acts as an anatomical prior — it knows where organs are likely to be from its training on medical text-image pairs. By constraining perturbations to these regions, attribution scores concentrate on boundary-relevant features the segmentation model actually uses, producing cleaner maps with fewer artifacts. The approach is model-agnostic: it wraps around any segmentation model (MedSAM, U-Net, nnU-Net) and works with any perturbation method (Occlusion, LIME, RISE).

## Step-by-Step Workflow

1. **Preprocess the medical image** with adaptive contrast enhancement: apply percentile-based windowing (5th/95th percentile bounds), then CLAHE (clip limit = 2.0 * N_pixels / 256, 8x8 tile grid) to normalize intensity variation across scanners and protocols.

2. **Load the vision-language model** (MediCLIP or BiomedCLIP) and define anatomical text prompts for each target organ — e.g., `["Liver", "Right Kidney", "Left Kidney", "Spleen"]`. If using Context Optimization (CoOp), prepend learnable token vectors `[V1][V2]...[Vm]` before the class label.

3. **Generate the ROI mask** by encoding the image and each text prompt through the CLIP vision and text encoders, computing cosine similarity at each spatial position, applying Gaussian smoothing to the resulting similarity map, and binarizing with threshold >= 0.5. Union all per-organ masks into a single binary ROI mask.

4. **Load the segmentation model** (MedSAM, U-Net, etc.) and compute the baseline segmentation prediction and its Dice score against ground truth (or a reference prediction if no ground truth is available).

5. **Configure the perturbation method:**
   - **Occlusion Sensitivity:** 64x64 patches, stride 32, zero-fill occlusion. Iterate only over patches where `ROI_mask[patch_region].any() == True`.
   - **RISE:** Generate 2,000 random binary masks at 7x7 base resolution with density p=0.5, upsample to input resolution via bilinear interpolation, then element-wise multiply each mask with the ROI mask before applying.
   - **LIME:** Run Felzenszwalb superpixel segmentation (scale=100, sigma=0.5, min_size=50), discard superpixels with zero overlap with ROI mask, sample 200-500 perturbation combinations from remaining superpixels.

6. **Run the perturbation loop:** For each perturbation, apply it to the input image, run the segmentation model forward pass, compute the Dice score change relative to baseline. Record the attribution score (Dice drop) for each patch/superpixel/mask.

7. **Accumulate the saliency map:** Aggregate per-perturbation attribution scores into a 2D heatmap. For occlusion, each patch writes its Dice-drop value. For RISE, weight predictions by mask correlation. For LIME, fit a linear surrogate model on superpixel features. Apply min-max normalization to [0, 1].

8. **Post-process and visualize:** Overlay the normalized saliency map on the original image using a diverging colormap (e.g., `jet` or `inferno`). Optionally threshold at 0.3-0.5 to highlight only high-attribution regions. Generate side-by-side panels: original image, ROI mask, saliency map, segmentation prediction.

9. **Compute evaluation metrics:** Report wall-clock latency, ROI reduction ratio (rho = N_patches_evaluated / N_patches_full), Dice fidelity of the saliency map, and IoU between high-attribution regions and ground-truth organ boundaries.

10. **Iterate across slices/volumes:** For 3D medical volumes, process each 2D slice independently, then stack saliency maps into a 3D volume for volumetric visualization.

## Concrete Examples

**Example 1: Occlusion-based explainability for liver segmentation**

User: "I have a MedSAM model segmenting livers on CT. I need to explain its predictions with occlusion sensitivity, but it's too slow — takes 45 seconds per slice."

Approach:
1. Load the CT slice and apply CLAHE preprocessing (clip_limit=2.0, tile_grid=8x8).
2. Run BiomedCLIP with prompt "Liver" to produce a binary ROI mask covering the liver region.
3. Configure occlusion: 64x64 patches, stride 32. Count total patches (~225 for 512x512). Count ROI-overlapping patches (typically 30-80 for a single organ).
4. Run MedSAM only on ROI-overlapping patches, recording Dice drop per patch.
5. Build the saliency heatmap from accumulated Dice drops.

Output:
```python
import numpy as np
from biomedclip import BiomedCLIPModel
from medsam import MedSAMPredictor

def xai_clip_occlusion(image, segmentation_model, clip_model, organ_prompt,
                       patch_size=64, stride=32, roi_threshold=0.5):
    # Step 1: Generate ROI mask via CLIP
    similarity_map = clip_model.get_similarity_map(image, organ_prompt)
    roi_mask = (gaussian_filter(similarity_map, sigma=2) >= roi_threshold).astype(np.uint8)

    # Step 2: Baseline segmentation
    baseline_pred = segmentation_model.predict(image)
    baseline_dice = compute_dice(baseline_pred, ground_truth)

    # Step 3: Selective occlusion
    h, w = image.shape[:2]
    saliency = np.zeros((h, w), dtype=np.float32)
    count = np.zeros((h, w), dtype=np.float32)

    for y in range(0, h - patch_size + 1, stride):
        for x in range(0, w - patch_size + 1, stride):
            patch_roi = roi_mask[y:y+patch_size, x:x+patch_size]
            if not patch_roi.any():
                continue  # Skip — no anatomical relevance
            occluded = image.copy()
            occluded[y:y+patch_size, x:x+patch_size] = 0
            pred = segmentation_model.predict(occluded)
            dice_drop = baseline_dice - compute_dice(pred, ground_truth)
            saliency[y:y+patch_size, x:x+patch_size] += dice_drop
            count[y:y+patch_size, x:x+patch_size] += 1

    saliency = np.divide(saliency, count, where=count > 0)
    saliency = (saliency - saliency.min()) / (saliency.max() - saliency.min() + 1e-8)
    return saliency, roi_mask
```

**Example 2: RISE with ROI masking for multi-organ explanation**

User: "Generate RISE-based saliency maps for kidney and spleen segmentation, constrained to anatomical regions."

Approach:
1. Generate per-organ ROI masks using CLIP prompts: `["Right Kidney", "Left Kidney", "Spleen"]`.
2. Union the masks into a combined ROI. Generate 2,000 RISE masks at 7x7 base resolution.
3. Element-wise AND each RISE mask with the ROI to zero out background perturbations.
4. For each masked perturbation, run the segmentation model and record predictions.
5. Compute weighted saliency as the correlation between mask presence and prediction confidence.

Output:
```python
def xai_clip_rise(image, seg_model, clip_model, organ_prompts,
                  n_masks=2000, base_res=7, density=0.5):
    # Generate union ROI mask from all organ prompts
    roi_mask = np.zeros(image.shape[:2], dtype=np.uint8)
    for prompt in organ_prompts:
        sim_map = clip_model.get_similarity_map(image, prompt)
        roi_mask |= (gaussian_filter(sim_map, sigma=2) >= 0.5).astype(np.uint8)

    h, w = image.shape[:2]
    saliency = np.zeros((h, w), dtype=np.float64)

    for _ in range(n_masks):
        # Generate random mask at low resolution, upsample
        base_mask = (np.random.rand(base_res, base_res) < density).astype(np.float32)
        mask = cv2.resize(base_mask, (w, h), interpolation=cv2.INTER_LINEAR)
        # Constrain to ROI
        mask = mask * roi_mask
        masked_image = image * mask[..., np.newaxis]
        pred = seg_model.predict(masked_image)
        score = compute_dice(pred, ground_truth)
        saliency += score * mask

    saliency /= n_masks * density
    saliency = (saliency - saliency.min()) / (saliency.max() - saliency.min() + 1e-8)
    return saliency
```

**Example 3: LIME with superpixel ROI filtering**

User: "Apply LIME to explain a U-Net pancreas segmentation, but only on relevant anatomy."

Approach:
1. Generate ROI mask via CLIP prompt "Pancreas".
2. Compute Felzenszwalb superpixels (scale=100, sigma=0.5, min_size=50).
3. Discard superpixels with <10% overlap with ROI mask.
4. Sample 300 binary perturbation vectors over remaining superpixels.
5. Fit a Ridge regression surrogate mapping superpixel presence to Dice score.
6. Extract per-superpixel coefficients as attribution weights.

Output:
```python
from skimage.segmentation import felzenszwalb
from sklearn.linear_model import Ridge

def xai_clip_lime(image, seg_model, clip_model, organ_prompt,
                  n_samples=300, min_overlap=0.1):
    roi_mask = (gaussian_filter(
        clip_model.get_similarity_map(image, organ_prompt), sigma=2) >= 0.5)
    segments = felzenszwalb(image, scale=100, sigma=0.5, min_size=50)
    unique_segs = np.unique(segments)

    # Filter superpixels by ROI overlap
    valid_segs = []
    for seg_id in unique_segs:
        seg_pixels = (segments == seg_id)
        overlap = (seg_pixels & roi_mask).sum() / seg_pixels.sum()
        if overlap >= min_overlap:
            valid_segs.append(seg_id)

    # Perturbation sampling
    features, scores = [], []
    for _ in range(n_samples):
        binary_vec = np.random.binomial(1, 0.5, size=len(valid_segs))
        perturbed = image.copy()
        for i, seg_id in enumerate(valid_segs):
            if binary_vec[i] == 0:
                perturbed[segments == seg_id] = 0
        pred = seg_model.predict(perturbed)
        features.append(binary_vec)
        scores.append(compute_dice(pred, ground_truth))

    model = Ridge(alpha=1.0).fit(np.array(features), np.array(scores))
    # Map coefficients back to image space
    saliency = np.zeros_like(image[:, :, 0], dtype=np.float64)
    for i, seg_id in enumerate(valid_segs):
        saliency[segments == seg_id] = model.coef_[i]
    saliency = (saliency - saliency.min()) / (saliency.max() - saliency.min() + 1e-8)
    return saliency
```

## Best Practices

- **Do:** Use organ-specific text prompts for ROI generation rather than generic prompts like "abnormal region" — specificity yields tighter, more accurate ROI masks.
- **Do:** Apply Gaussian smoothing (sigma=2-3) to the CLIP similarity map before binarization to avoid fragmented ROI masks with holes.
- **Do:** Preserve non-ROI pixels at their original value (multiplier 1.0) during perturbation — zeroing the entire background conflates region importance with context dependency.
- **Do:** Validate the ROI mask visually before running the perturbation loop — a bad ROI mask wastes the entire computation.
- **Avoid:** Setting the ROI binarization threshold too low (< 0.3), which expands the mask to include irrelevant tissue and negates the computational savings.
- **Avoid:** Using very small patch sizes (< 32x32) for occlusion — they increase patch count quadratically and produce overly granular, noisy attribution maps without proportional interpretability gains.
- **Avoid:** Running RISE without ROI masking on medical images — unmasked random perturbations in background regions dominate the saliency signal and obscure organ-boundary attributions.

## Error Handling

| Problem | Cause | Fix |
|---------|-------|-----|
| ROI mask is empty (all zeros) | CLIP model has low confidence for the target organ, or the organ is absent from the slice | Fall back to a bounding-box ROI from the segmentation prediction itself, or skip the slice |
| ROI mask covers entire image | Threshold too low or image lacks contrast | Increase threshold to 0.6-0.7, verify CLAHE preprocessing was applied |
| Saliency map is uniform/flat | Segmentation model is robust to occlusion in this region, or patch size is too large | Reduce patch size, increase perturbation intensity, or try LIME instead of occlusion |
| Out-of-memory on GPU | Too many RISE masks or volume is 3D | Process masks in batches of 100-200, process 2D slices independently |
| Saliency bleeds across organ boundaries | Patch stride too large relative to organ size | Decrease stride (use stride = patch_size / 2), or switch to superpixel-based LIME |

## Limitations

- **CLIP model dependency:** The quality of ROI masks is bounded by the vision-language model's training data. Rare pathologies or unusual anatomy (e.g., situs inversus) may produce inaccurate ROI localization.
- **2D slice-based:** The pipeline operates on 2D slices. Volumetric coherence across slices is not enforced — adjacent slices may have inconsistent saliency patterns.
- **Not a faithfulness guarantee:** ROI-guided perturbation improves efficiency and map quality, but does not prove the segmentation model uses the highlighted features. It remains a post-hoc approximation.
- **Organ-specific prompts required:** Each target anatomy needs a corresponding text prompt. For novel or unnamed structures, prompt engineering or fine-tuning (CoOp) is necessary.
- **Preprocessing sensitivity:** CLAHE parameters and percentile windowing must be tuned per imaging modality (CT vs. MRI vs. ultrasound). A single configuration does not generalize across modalities.

## Reference

- **Paper:** [XAI-CLIP: ROI-Guided Perturbation Framework for Explainable Medical Image Segmentation in Multimodal Vision-Language Models](https://arxiv.org/abs/2602.07017v1) — Look for Section 3 (Methodology) for the full pipeline architecture, Section 3.2 for the MediCLIP ROI extraction details, and Tables 2-4 for quantitative comparisons across FLARE22 and CHAOS datasets.