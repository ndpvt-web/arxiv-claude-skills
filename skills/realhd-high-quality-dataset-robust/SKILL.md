---
name: "realhd-high-quality-dataset-robust"
description: "Detect AI-generated images using NLM noise entropy analysis and build robust forensic detection pipelines. Use when: 'detect if an image is AI-generated', 'build an AI image detector', 'classify real vs fake images', 'image forensics pipeline', 'noise entropy feature extraction', 'detect deepfakes or inpainted regions'."
---

# RealHD: AI-Generated Image Detection via NLM Noise Entropy

This skill enables Claude to build image forensic detection systems that distinguish AI-generated images from authentic photographs. The core technique extracts a residual noise map using Non-Local Means (NLM) denoising, then computes block-wise Shannon entropy over the noise to produce a compact entropy tensor. This entropy tensor replaces raw pixels as input to a classifier (ResNet-50, Xception, or EfficientFormer), yielding strong cross-generator generalization — models trained with this method on one set of generators transfer well to unseen generators, outperforming pixel-only baselines by significant margins.

## When to Use

- When the user asks to **detect whether an image was AI-generated** (by DALL-E, Stable Diffusion, Midjourney, FLUX, etc.)
- When building an **image forensics or authenticity verification pipeline** for a product or research project
- When the user needs to **extract noise-based features** from images for classification
- When designing a **training dataset** for AI-generated image detection, including text-to-image, inpainting, refinement, and face-swap categories
- When the user wants to **evaluate detector robustness** against JPEG compression or unseen generators
- When implementing **inpainting localization** — detecting which regions of an image were modified

## Key Technique: NLM Noise Entropy Transform

**Why noise, not pixels?** AI generators leave subtle statistical fingerprints in image noise patterns. Real camera sensors produce characteristic noise (thermal, shot, read noise), while generative models produce noise with different entropy distributions. By isolating the noise and measuring its information content, you get a feature space where real and fake images are more separable than in pixel space — and this transfers across generators.

**The NLM Noise Entropy pipeline has three stages:**

1. **Noise extraction via Non-Local Means denoising.** Given an input image `x`, compute the NLM-denoised version `x_hat` where each pixel is a weighted average of similar patches in a search window. The residual noise map is `eta = x - x_hat`. NLM is chosen over simpler denoisers (Gaussian blur, median filter) because it preserves edges while removing structured content, leaving a cleaner noise residual. Key NLM parameters: patch size (typically 7x7), search window (21x21), filter strength `h` (controls denoising aggressiveness).

2. **Block-wise Shannon entropy computation.** Resize the noise tensor to 3 x 1024 x 1024 (three RGB channels). Partition each channel independently into n x n non-overlapping blocks. For each block, quantize pixel values into `K` bins and compute Shannon entropy: `H(i,j) = -sum(p_k * log(p_k))` where `p_k` is the normalized frequency of bin `k`. This produces an entropy map of shape `(3, n, n)` per channel.

3. **Tensor formation and classification.** Concatenate the three channel entropy maps into a final entropy tensor `H in R^{3 x n x n}`. This compact representation (e.g., 3x32x32 = 3072 values vs. 3x224x224 = 150,528 pixels) is fed to a standard image classifier. The paper uses ResNet-50, Xception, and EfficientFormer, trained with Adam (lr=0.00125, batch size 64, 15 epochs). Images are resized to 224x224 for pixel-based baselines, but the entropy tensor itself is the n x n grid resolution.

## Step-by-Step Workflow

1. **Install dependencies.** Set up OpenCV (for NLM denoising via `cv2.fastNlMeansDenoisingColored`), NumPy, PyTorch/torchvision, and scipy. Pin OpenCV >= 4.5 for consistent NLM behavior.

2. **Load and preprocess the input image.** Read the image in BGR (OpenCV) or RGB, resize to 1024x1024 (or the working resolution), and convert to float32 for precision. Keep the original for display.

3. **Extract the NLM noise residual.** Apply `cv2.fastNlMeansDenoisingColored(img, None, h=10, hForColorComponents=10, templateWindowSize=7, searchWindowSize=21)` to get the denoised image. Subtract: `noise = img.astype(np.float32) - denoised.astype(np.float32)`. The result is a 3-channel noise map.

4. **Compute block-wise entropy.** Divide each noise channel into an n x n grid of non-overlapping blocks (e.g., n=32 gives 32x32 pixel blocks from a 1024x1024 image). For each block, compute a histogram with K bins (e.g., K=256), normalize to a probability distribution, and calculate Shannon entropy. Store results in a tensor of shape `(3, n, n)`.

5. **Normalize the entropy tensor.** Scale entropy values to [0, 1] range per channel using min-max normalization. This stabilizes training across images with different noise magnitudes.

6. **Build or load the classifier.** Use a lightweight backbone — EfficientFormer-L1 gives the best accuracy/speed tradeoff per the paper. Modify the first convolutional layer to accept the entropy tensor dimensions if needed, or resize the entropy tensor to match the model's expected input (e.g., 224x224 via bilinear interpolation).

7. **Train the binary classifier (real vs. generated).** Use binary cross-entropy loss, Adam optimizer (lr=0.00125), batch size 64, for 15 epochs. Apply standard augmentation (horizontal flip, slight rotation) on the original image *before* noise extraction — augment then extract, not extract then augment.

8. **Evaluate with cross-dataset generalization.** Test on images from generators not seen during training. Report accuracy and AUC. For robustness testing, apply JPEG compression at quality factors 90, 75, and 50 before noise extraction and measure degradation.

9. **For inpainting detection, add localization.** If the goal includes localizing manipulated regions, train a segmentation head on the entropy tensor. Use the binary masks from inpainting datasets as ground truth. The entropy map naturally highlights regions with different noise statistics.

10. **Deploy as inference pipeline.** Wrap steps 2-6 into a single `predict(image_path) -> {label, confidence}` function. NLM denoising is the bottleneck (~100ms per 1024x1024 image on CPU); consider GPU-accelerated denoising for production throughput.

## Concrete Examples

**Example 1: Build a real-vs-fake image classifier**

User: "I have a folder of images and want to detect which ones are AI-generated. Can you build a detection pipeline?"

Approach:
1. Create a Python module with three components: `noise_extractor.py`, `entropy_transform.py`, and `classifier.py`.
2. In `noise_extractor.py`, implement NLM denoising and residual computation:

```python
import cv2
import numpy as np

def extract_nlm_noise(image_path: str, h: float = 10.0) -> np.ndarray:
    """Extract NLM noise residual from an image."""
    img = cv2.imread(image_path)
    img = cv2.resize(img, (1024, 1024))
    denoised = cv2.fastNlMeansDenoisingColored(
        img, None, h, h,
        templateWindowSize=7,
        searchWindowSize=21
    )
    noise = img.astype(np.float32) - denoised.astype(np.float32)
    return noise  # shape: (1024, 1024, 3)
```

3. In `entropy_transform.py`, compute block-wise Shannon entropy:

```python
import numpy as np
from scipy.stats import entropy as shannon_entropy

def compute_entropy_tensor(noise: np.ndarray, n_blocks: int = 32, n_bins: int = 256) -> np.ndarray:
    """Convert noise map to entropy tensor of shape (3, n_blocks, n_blocks)."""
    H, W, C = noise.shape
    block_h, block_w = H // n_blocks, W // n_blocks
    entropy_tensor = np.zeros((C, n_blocks, n_blocks), dtype=np.float32)

    for c in range(C):
        for i in range(n_blocks):
            for j in range(n_blocks):
                block = noise[i*block_h:(i+1)*block_h, j*block_w:(j+1)*block_w, c]
                hist, _ = np.histogram(block, bins=n_bins, density=True)
                hist = hist[hist > 0]
                entropy_tensor[c, i, j] = shannon_entropy(hist, base=2)

    # Min-max normalize per channel
    for c in range(C):
        ch = entropy_tensor[c]
        ch_min, ch_max = ch.min(), ch.max()
        if ch_max > ch_min:
            entropy_tensor[c] = (ch - ch_min) / (ch_max - ch_min)
    return entropy_tensor
```

4. In `classifier.py`, wrap a pretrained backbone:

```python
import torch
import torch.nn as nn
from torchvision.models import resnet50

class NoiseEntropyClassifier(nn.Module):
    def __init__(self, n_blocks: int = 32):
        super().__init__()
        self.upsample = nn.Upsample(size=(224, 224), mode='bilinear', align_corners=False)
        backbone = resnet50(weights=None)
        backbone.fc = nn.Linear(backbone.fc.in_features, 1)
        self.backbone = backbone

    def forward(self, entropy_tensor: torch.Tensor) -> torch.Tensor:
        x = self.upsample(entropy_tensor)  # (B, 3, 224, 224)
        return torch.sigmoid(self.backbone(x))
```

Output: A `predict()` function returning `{"label": "ai_generated", "confidence": 0.94}` or `{"label": "real", "confidence": 0.87}`.

---

**Example 2: Evaluate detector robustness to JPEG compression**

User: "My images get JPEG-compressed when uploaded to social media. Will the detector still work?"

Approach:
1. Create a JPEG simulation pipeline that compresses images at quality factors 90, 75, and 50 before feeding them to the NLM entropy pipeline.
2. Measure AUC degradation at each quality level.

```python
import cv2
import numpy as np

def simulate_jpeg(image_path: str, quality: int) -> np.ndarray:
    """Apply JPEG compression to an image and return the result."""
    img = cv2.imread(image_path)
    encode_param = [int(cv2.IMWRITE_JPEG_QUALITY), quality]
    _, encoded = cv2.imencode('.jpg', img, encode_param)
    return cv2.imdecode(encoded, cv2.IMREAD_COLOR)

# Expected AUC degradation pattern (from paper):
# Q=90: AUC ~0.77 (moderate drop)
# Q=75: AUC ~0.67 (significant drop)
# Q=50: AUC ~0.59 (severe drop, noise signal destroyed)
```

3. Advise the user: JPEG compression destroys high-frequency noise information. For social media use cases, train with JPEG-augmented data — apply random JPEG compression (Q=70-95) during training to build robustness.

---

**Example 3: Build a dataset for AI-generated image detection**

User: "I want to create my own training dataset for detecting AI-generated images. How should I structure it?"

Approach:
1. Follow the RealHD category taxonomy — organize by content type (portrait, landscape, art, animal, news) and by generation method.
2. Source real images from verified datasets (museum collections, news agencies, curated photo datasets) to avoid contamination.
3. Generate synthetic images across multiple methods:

```
dataset/
  real/
    portrait/       # ~100K images from verified photo datasets
    landscape/      # ~15K from curated landscape collections
    art/            # ~55K from museum/gallery databases
    animal/         # ~11K from wildlife photo datasets
    news/           # ~22K from verified news agencies
  generated/
    text_to_image/
      sd_v1.5/      # Stable Diffusion 1.5
      sd_v2.1/      # Stable Diffusion 2.1
      sd_v3.0/      # Stable Diffusion 3.0
      sdxl/         # Stable Diffusion XL
      flux/         # FLUX models
    inpainting/
      sdxl_inpaint/ # With binary masks in masks/ subfolder
      sd2_inpaint/
    refinement/
      sdxl_refiner/ # 512->1024 upscaled images
    face_swap/
      face_adapter/ # Source-target face swaps
  metadata/
    annotations.json  # {filename, method, category, generator, prompt}
```

4. Write diverse prompts — at least 10,000 unique prompts with rich semantic content. Avoid simple prompts like "a cat" — use detailed descriptions: "a tabby cat sitting on a weathered wooden fence at golden hour, shallow depth of field, film grain."
5. Save images as PNG or JPEG at quality >= 90 to preserve noise characteristics.

## Best Practices

- **Do:** Always extract noise at the highest available image resolution (ideally 1024x1024) before any downsampling. NLM noise patterns are resolution-sensitive.
- **Do:** Train with images from at least 4-5 different generators. The paper shows that multi-generator training is the key to cross-generator generalization.
- **Do:** Include both fully synthetic (text-to-image) and partially manipulated (inpainting, face-swap) images in training data. Detectors trained only on fully synthetic images miss local manipulations.
- **Do:** Use `h=10` as the default NLM filter strength. Lower values leave too much structure in the residual; higher values remove genuine noise signal.
- **Avoid:** Training or evaluating on heavily JPEG-compressed images (Q < 75) without compression-aware augmentation. JPEG destroys the high-frequency noise that NLM entropy depends on.
- **Avoid:** Using only accuracy as your metric. Always report AUC alongside accuracy — class-imbalanced datasets make accuracy misleading.

## Error Handling

- **NLM is slow on large images.** For images > 2048px, downsample to 1024x1024 first. NLM has O(n * search_window^2 * patch_size^2) complexity per pixel. On CPU, expect ~100-500ms per 1024x1024 image.
- **Entropy of constant blocks is zero.** Blocks that fall entirely within flat regions (sky, solid backgrounds) produce zero entropy. This is valid signal — don't replace zeros. Flat regions in real photos have sensor noise; flat regions in AI images are truly flat.
- **OpenCV NLM produces uint8 output.** Cast both `img` and `denoised` to float32 before subtraction to avoid underflow. Negative noise values are meaningful and must be preserved.
- **Histogram bin count affects entropy resolution.** Using too few bins (< 64) loses noise detail; too many bins (> 512) produces sparse histograms with unreliable entropy estimates. Default to 256 bins.
- **Model input mismatch.** If using a pretrained backbone expecting 224x224 RGB input, the 3xNxN entropy tensor must be resized via bilinear interpolation. Do not use nearest-neighbor — it creates artifacts in the entropy map.

## Limitations

- **JPEG compression severely degrades performance.** At quality factor 50, AUC drops to ~0.59 (near random). This method works best on high-quality PNG or minimally-compressed JPEG images.
- **Generalization to future generators is not guaranteed.** The method generalizes well across current diffusion models but may need retraining as generation techniques evolve (e.g., autoregressive image models, consistency models).
- **NLM denoising is computationally expensive** compared to pixel-based methods. Not suitable for real-time video analysis without GPU acceleration or algorithmic approximations.
- **Small images (< 256px) produce unreliable entropy tensors.** With n=32 blocks, each block is only 8x8 pixels — too few samples for stable entropy estimation. Minimum recommended input: 512x512.
- **Does not explain *why* an image is fake.** The entropy tensor provides a discriminative signal but no human-interpretable explanation. For explainability, visualize the entropy heatmap — regions with anomalous entropy often correspond to generated or manipulated areas.
- **Single-image detection only.** This method analyzes individual images. It does not detect provenance, chains of edits, or multi-image forgeries.

## Reference

**Paper:** Yu et al., "RealHD: A High-Quality Dataset for Robust Detection of State-of-the-Art AI-Generated Images," ACM MM 2025. [arXiv:2602.10546](https://arxiv.org/abs/2602.10546v1)

Key sections to study: Section 3 (Dataset Construction) for the multi-method generation pipeline and category taxonomy; Section 4 (NLM Noise Entropy method) for the mathematical formulation of noise extraction, block entropy, and tensor construction; Table 3 (cross-dataset generalization results) for understanding why multi-generator training matters.