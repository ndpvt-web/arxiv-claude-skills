---
name: "multimodal-learning-arcing-detection"
description: "Build multimodal anomaly detection systems that fuse image and sensor data using the MultiDeepSAD framework — a semi-supervised deep approach with modality-specific pseudo-anomaly generation. Use when asked to: 'detect anomalies from images and sensor data', 'build a multimodal anomaly detector', 'fuse visual and time-series data for defect detection', 'implement DeepSAD with multiple modalities', 'generate pseudo-anomalies for training data augmentation', 'detect rare events from combined camera and sensor feeds'."
---

# Multimodal Anomaly Detection with MultiDeepSAD

This skill enables Claude to implement multimodal semi-supervised anomaly detection systems that combine image data with time-series sensor measurements. The core technique, MultiDeepSAD, extends the Deep Semi-supervised Anomaly Detection (DeepSAD) algorithm to handle multiple data modalities through late fusion of modality-specific encoders, an exponential loss reformulation for stable training, and targeted pseudo-anomaly generation per modality. This approach is especially powerful when real anomalies are scarce (as few as 1-2 labeled examples) and the system must generalize across domain shifts.

## When to Use

- When the user wants to detect rare events or anomalies from paired image and sensor/signal data (e.g., industrial inspection with cameras + vibration sensors)
- When building a semi-supervised anomaly detector where normal data is abundant but labeled anomalies are extremely scarce
- When the user needs to fuse visual features (CNN-based) with time-series or frequency-domain features (MLP-based) for a joint detection model
- When the user asks about pseudo-anomaly or synthetic anomaly generation to augment limited training data
- When implementing one-class or semi-supervised deep learning for manufacturing defect detection, infrastructure monitoring, or equipment fault detection
- When the user needs an anomaly detection system robust to domain shifts (e.g., different lighting, weather, or sensor drift)

## Key Technique

**MultiDeepSAD** is a semi-supervised anomaly detection framework that maps multimodal inputs into a shared hypersphere embedding space. Normal samples are pulled toward a learned center point `c`, while known anomalies are pushed away. The architecture uses modality-specific encoders — a ResNet-18 for images and a two-layer MLP operating on FFT-transformed signals for time-series data — whose outputs are concatenated (late fusion) and passed through a fusion MLP. The key innovation over vanilla DeepSAD is the loss reformulation: instead of an inverse-distance penalty for anomalies (which causes gradient explosion), MultiDeepSAD uses an exponential penalty `exp(-||phi(x) - c||^2)` that provides smoother gradients and stable optimization, especially when anomalies are near the decision boundary.

**Pseudo-anomaly generation** addresses the fundamental data scarcity problem. For images, real arcing/defect regions are cropped from the few available anomalous samples and pasted onto normal images at plausible locations, creating synthetic anomalies that preserve visual realism. For time-series signals, a Mixup-inspired approach generates convex combinations `d = lambda * a_abnormal + (1-lambda) * a_normal` with lambda drawn from a Beta distribution, producing signals that interpolate between normal and abnormal characteristics. These modality-aware augmentations are critical — generic augmentation methods like standard Mixup perform significantly worse (90.2% vs. 93.5% AUROC).

**Late fusion** outperforms more complex alternatives. The paper's ablation shows that simple concatenation of modality embeddings followed by an MLP beats weighted fusion, gated fusion, and attention-based fusion. This is a practical insight: start simple with late fusion before adding complexity.

## Step-by-Step Workflow

1. **Define the modality-specific data pipeline.** Separate image data from sensor/signal data. For images, apply standard preprocessing (resize, normalize to ImageNet stats). For time-series signals, apply FFT and retain the first half of the normalized magnitude spectrum as the frequency-domain representation.

2. **Build modality-specific encoders.** Use a ResNet-18 (pretrained on ImageNet) as the image encoder, removing the final classification layer to output a feature vector. Build a two-layer MLP for the signal encoder, taking FFT magnitudes as input. Pretrain the signal encoder as an autoencoder with reconstruction loss for ~100 epochs before joint training.

3. **Implement the late fusion module.** Concatenate the output embeddings from both encoders: `z = [z_image; z_signal]`. Pass the concatenated vector through a two-layer MLP to produce the final fused embedding `phi(x)`.

4. **Compute the hypersphere center `c`.** Run a forward pass over the entire normal training set and compute the mean of all fused embeddings. Fix `c` as this mean (do not update during training). Clamp any near-zero dimensions of `c` away from zero to avoid mode collapse.

5. **Implement the MultiDeepSAD loss.**
   ```python
   # For normal samples: minimize distance to center
   loss_normal = torch.mean(torch.sum((phi_normal - c) ** 2, dim=1))
   # For anomaly/pseudo-anomaly samples: exponential repulsion
   loss_anomaly = torch.mean(torch.exp(-torch.sum((phi_anomaly - c) ** 2, dim=1)))
   # Combined loss
   loss = (1 / n_normal) * loss_normal + eta * (1 / n_anomaly) * loss_anomaly
   ```
   Set `eta = 1` as the weighting hyperparameter.

6. **Generate pseudo-anomalies for images.** Crop defect/anomaly regions from the few labeled anomalous images. For each training batch, randomly select normal images and paste cropped anomaly patches at plausible spatial locations (e.g., near the region of interest). Apply blending or feathering at patch boundaries for realism.

7. **Generate pseudo-anomalies for signals.** Use the Mixup approach with Beta distribution sampling:
   ```python
   lam = np.random.beta(alpha, alpha)  # alpha ~ 0.5-2.0
   synthetic_signal = lam * real_abnormal_signal + (1 - lam) * normal_signal
   ```
   Label these synthetic signals as anomalous for training.

8. **Train the joint model.** Use Adam optimizer with lr=1e-4, batch size 64, for 30 epochs. Each batch should contain normal samples, real anomalies (if available), and pseudo-anomalies from both modalities. Ensure paired image-signal samples are kept synchronized.

9. **Score anomalies at inference.** Compute the squared distance `||phi(x) - c||^2` for each test sample. Higher distance = more anomalous. Set a threshold based on validation data or use AUROC for evaluation.

10. **Evaluate robustness.** Test under domain shift conditions (different lighting, noise levels, sensor drift). The multimodal approach provides resilience — when one modality degrades (e.g., fog corrupting images), the other modality (signals) compensates.

## Concrete Examples

**Example 1: Industrial weld defect detection with camera + ultrasonic sensor**

User: "I have a dataset of weld images paired with ultrasonic sensor readings. Most welds are normal, but I have ~5 labeled defective welds. Build an anomaly detector."

Approach:
1. Preprocess images: resize to 224x224, normalize with ImageNet mean/std
2. Preprocess ultrasonic signals: apply FFT, keep first N/2 magnitude bins, normalize
3. Build encoders: ResNet-18 for images (output 512-d), 2-layer MLP for FFT features (output 128-d)
4. Fuse: concatenate [512 + 128] = 640-d, pass through MLP -> 128-d embedding
5. Generate pseudo-anomalies: crop defect regions from 5 labeled samples, paste onto normal weld images; Mixup normal and abnormal ultrasonic signals
6. Train with MultiDeepSAD loss for 30 epochs

Output:
```python
import torch
import torch.nn as nn
from torchvision import models

class ImageEncoder(nn.Module):
    def __init__(self, embed_dim=512):
        super().__init__()
        resnet = models.resnet18(pretrained=True)
        self.backbone = nn.Sequential(*list(resnet.children())[:-1])
        self.fc = nn.Linear(512, embed_dim)

    def forward(self, x):
        x = self.backbone(x).squeeze(-1).squeeze(-1)
        return self.fc(x)

class SignalEncoder(nn.Module):
    def __init__(self, input_dim, embed_dim=128):
        super().__init__()
        self.net = nn.Sequential(
            nn.Linear(input_dim, 256), nn.ReLU(),
            nn.Linear(256, embed_dim), nn.ReLU(),
        )

    def forward(self, x):
        return self.net(x)

class MultiDeepSAD(nn.Module):
    def __init__(self, img_embed=512, sig_embed=128, fused_dim=128):
        super().__init__()
        self.image_enc = ImageEncoder(img_embed)
        self.signal_enc = SignalEncoder(input_dim=256, embed_dim=sig_embed)
        self.fusion = nn.Sequential(
            nn.Linear(img_embed + sig_embed, 256), nn.ReLU(),
            nn.Linear(256, fused_dim),
        )

    def forward(self, images, signals):
        z_img = self.image_enc(images)
        z_sig = self.signal_enc(signals)
        z_fused = self.fusion(torch.cat([z_img, z_sig], dim=1))
        return z_fused

def multi_deep_sad_loss(embeddings, labels, center, eta=1.0):
    """labels: 0=normal, 1=anomaly/pseudo-anomaly"""
    dists = torch.sum((embeddings - center) ** 2, dim=1)
    normal_mask = labels == 0
    anomaly_mask = labels == 1
    loss_n = dists[normal_mask].mean() if normal_mask.any() else 0
    loss_a = torch.exp(-dists[anomaly_mask]).mean() if anomaly_mask.any() else 0
    return loss_n + eta * loss_a
```

**Example 2: Power line monitoring with thermal camera + current sensors**

User: "We monitor overhead power lines with thermal cameras and current sensors. Arcing/flashover events are rare. Can you build a detector that works with very few labeled examples?"

Approach:
1. Apply FFT to current sensor waveforms, extract magnitude spectrum
2. Use ResNet-18 on thermal images, MLP on FFT features
3. Since labeled anomalies are extremely scarce (1-2 samples), pseudo-anomaly generation is critical:
   - Crop bright arc regions from labeled thermal images, overlay on normal frames at conductor contact points
   - Generate synthetic current spikes via Mixup of the 1-2 abnormal waveforms with normal ones
4. Train MultiDeepSAD — with just 2 real anomalies + pseudo-anomalies, expect ~93% AUROC

Output:
```python
import numpy as np

def generate_image_pseudo_anomalies(normal_images, anomaly_crops, n_synthetic=500):
    """Paste anomaly region crops onto normal images at random valid positions."""
    synthetic = []
    for _ in range(n_synthetic):
        base = normal_images[np.random.randint(len(normal_images))].copy()
        crop = anomaly_crops[np.random.randint(len(anomaly_crops))]
        # Random position within region of interest
        max_y = base.shape[0] - crop.shape[0]
        max_x = base.shape[1] - crop.shape[1]
        y, x = np.random.randint(0, max(max_y, 1)), np.random.randint(0, max(max_x, 1))
        # Alpha blending for realism
        alpha = 0.8
        region = base[y:y+crop.shape[0], x:x+crop.shape[1]]
        base[y:y+crop.shape[0], x:x+crop.shape[1]] = (
            alpha * crop + (1 - alpha) * region
        ).astype(base.dtype)
        synthetic.append(base)
    return synthetic

def generate_signal_pseudo_anomalies(normal_signals, abnormal_signals,
                                      n_synthetic=500, alpha=1.0):
    """Mixup-based synthetic anomaly generation for time-series."""
    synthetic = []
    for _ in range(n_synthetic):
        norm = normal_signals[np.random.randint(len(normal_signals))]
        abn = abnormal_signals[np.random.randint(len(abnormal_signals))]
        lam = np.random.beta(alpha, alpha)
        synthetic.append(lam * abn + (1 - lam) * norm)
    return np.array(synthetic)
```

**Example 3: Adapting to a single-modality scenario**

User: "I only have images, no sensor data. Can I still use this approach?"

Approach:
1. Use only the image encoder branch (ResNet-18) with the same DeepSAD-style loss
2. Apply the exponential loss reformulation — this alone improves over standard DeepSAD (the paper shows image-only MultiDeepSAD loss outperforms original DeepSAD loss)
3. Use pseudo-anomaly generation on images (crop-and-paste)
4. Expect ~89% AUROC with images alone vs. ~93% with both modalities — still a strong baseline

## Best Practices

- **Do:** Pretrain the signal/time-series encoder as an autoencoder before joint training. The paper shows 100 epochs of autoencoder pretraining on normal signal data significantly improves the quality of signal embeddings.
- **Do:** Use FFT as the default signal representation. It outperformed wavelet transforms, STFT, Gramian Angular Fields, Markov Transition Fields, and recurrence plots in ablation studies.
- **Do:** Start with late fusion (concatenation + MLP). It consistently outperformed attention fusion, gated fusion, and weighted fusion in experiments.
- **Do:** Fix the hypersphere center `c` after computing it from the initial forward pass on training data. Do not make it a learnable parameter — this prevents mode collapse where the network trivially maps everything to zero.
- **Avoid:** Using the original DeepSAD inverse penalty `1/||phi(x) - c||^2` for anomalies. It causes gradient explosion when anomaly embeddings are near the center. Always use the exponential formulation `exp(-||phi(x) - c||^2)`.
- **Avoid:** Generic data augmentation (standard Mixup, CutMix) applied uniformly across modalities. Modality-specific pseudo-anomaly strategies (crop-paste for images, Mixup for signals) yield meaningfully better results (93.5% vs. 90.2% AUROC).

## Error Handling

- **Gradient explosion during training:** If loss diverges, verify you are using the exponential anomaly loss, not the inverse formulation. Reduce learning rate to 1e-5 and check that center `c` has no near-zero dimensions (clamp to epsilon=0.01).
- **Mode collapse (all embeddings map to same point):** Ensure center `c` is fixed, not learned. Verify that the autoencoder pretraining for the signal encoder completed successfully before joint training.
- **Poor performance on one modality:** Check FFT preprocessing — signals must be normalized before FFT, and only the first N/2 magnitude bins should be retained. For images, verify ImageNet normalization is applied.
- **Pseudo-anomaly generation hurts performance:** If synthetic anomalies are too easy to distinguish from real ones (trivial augmentation), the model won't learn useful boundaries. Ensure image patches are pasted at semantically plausible locations and Mixup lambda values span a wide range (use Beta alpha between 0.5 and 2.0).
- **Domain shift degradation:** If deploying to a new environment (different lighting, sensor calibration), fine-tune with a small amount of normal data from the target domain. The multimodal approach is inherently more robust — fog severely degrades image-only detection (67% AUROC) but multimodal fusion recovers significantly.

## Limitations

- **Requires paired multimodal data.** Image and signal measurements must be synchronized in time. If only one modality is available, the framework still works but with reduced detection performance (~89% vs. ~93% AUROC).
- **Pseudo-anomaly quality depends on having at least 1-2 real anomaly examples.** With zero labeled anomalies, the crop-paste and Mixup strategies cannot be applied, and the system falls back to unsupervised one-class detection.
- **Late fusion assumes modality independence.** If complex cross-modal interactions exist (e.g., a specific image pattern only matters when combined with a specific signal pattern), late fusion may miss them. However, empirically it outperforms more complex fusion methods on the tested domains.
- **ResNet-18 image encoder may underperform on very high-resolution or fine-grained visual anomalies.** The paper found ResNet-18 slightly outperformed ResNet-50 and ViT-B/16, but this may not generalize to all domains.
- **Not designed for real-time streaming.** The framework processes discrete paired samples. Adapting it for continuous video + sensor streams requires windowing and synchronization logic not covered by the core method.

## Reference

[Multimodal Learning for Arcing Detection in Pantograph-Catenary Systems](https://arxiv.org/abs/2602.08792v1) — Dong, Chatzi, Fink (2026). Focus on Section 3 (MultiDeepSAD architecture and loss), Section 3.3 (pseudo-anomaly generation), and Tables 1-4 (ablation studies showing modality fusion, loss formulation, and augmentation comparisons).