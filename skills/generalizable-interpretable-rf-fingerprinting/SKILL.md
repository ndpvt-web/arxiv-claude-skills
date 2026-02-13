---
name: "generalizable-interpretable-rf-fingerprinting"
description: "Build RF fingerprinting systems that combine learnable 2D shapelets with pre-trained LLMs for wireless device authentication. Handles cross-domain generalization and few-shot device identification on I/Q signal data. Use when: 'authenticate IoT devices from RF signals', 'build wireless device fingerprinting', 'classify devices from I/Q data', 'few-shot RF device identification', 'cross-domain wireless authentication', 'interpretable RF feature extraction'."
---

This skill enables Claude to implement shapelet-enhanced LLM pipelines for radio frequency (RF) fingerprinting — authenticating wireless devices by their unique hardware-level signal characteristics. The core technique from Zhao et al. (2026) fuses variable-length 2D shapelets (learned discriminative subsequences over I/Q signal pairs) with a frozen-then-fine-tuned LLM backbone, yielding a system that is simultaneously interpretable (shapelets show *which* local signal patterns matter), generalizable (the LLM captures global context that transfers across environments), and few-shot capable (prototype-based inference works with as few as 1–5 examples per device).

## When to Use

- When the user needs to authenticate or identify wireless devices from raw I/Q (in-phase/quadrature) signal captures.
- When building an RF fingerprinting system that must generalize across different physical environments (rooms, distances, channel conditions) without full retraining.
- When the user wants interpretable device identification — needing to explain *why* a device was classified a certain way, not just the prediction.
- When implementing few-shot device enrollment, where new devices must be recognized from very few signal samples.
- When processing 2D time-series data (dual-channel signals like I/Q, stereo audio, or paired sensor streams) and standard 1D shapelet methods are insufficient.
- When combining time-series pattern mining with transformer/LLM feature extraction for any signal classification task.

## Key Technique

**2D Shapelets as Interpretable Local Features.** A shapelet is a short, learned subsequence that acts as a discriminative template. In standard time-series work, shapelets are 1D. This framework extends them to 2D: each shapelet is a short matrix of shape `(L_k, 2)` where `L_k` is a variable length and the two columns correspond to the I (in-phase) and Q (quadrature) components of the RF signal. A group of shapelets with different lengths captures diverse local temporal patterns — short shapelets detect transient features (e.g., turn-on transients), while longer ones capture oscillatory hardware imperfections. The distance between a shapelet and an I/Q window is computed via a sliding minimum-distance operation, producing a compact feature vector that is directly inspectable: you can visualize which signal region best matched each shapelet and understand the fingerprint.

**LLM Backbone for Global Context.** The I/Q signal is also segmented and projected into token embeddings compatible with a pre-trained LLM (e.g., GPT-2 or LLaMA). Most LLM layers are frozen to preserve their general sequence-modeling capability; only a lightweight adapter or the final few layers are fine-tuned. This branch captures long-range dependencies and global spectral structure that shapelets alone miss. The shapelet features and LLM features are fused (concatenation + projection) before the classification head.

**Prototype-Based Few-Shot Inference.** For cross-domain or new-device scenarios, the framework computes a class prototype for each device by averaging the fused feature vectors of available support samples. At inference, a query signal's feature vector is compared against all prototypes via cosine or Euclidean distance, and the nearest prototype determines the device identity. This eliminates retraining when deploying to a new environment or enrolling new devices.

## Step-by-Step Workflow

1. **Ingest and preprocess I/Q data.** Load raw I/Q signal captures into a 2D array of shape `(N, T, 2)` where `N` is the number of samples, `T` is the time length, and the last dimension holds I and Q channels. Normalize each channel to zero mean and unit variance per-sample. Segment long captures into fixed-length windows (e.g., 1024 or 2048 time steps) with optional overlap.

2. **Initialize a group of variable-length 2D shapelets.** Create `K` shapelets (e.g., K=16) with lengths sampled from a range (e.g., 32 to 256 time steps). Initialize each shapelet tensor of shape `(L_k, 2)` from random signal segments or via k-means clustering on signal windows. Register them as learnable `nn.Parameter` tensors.

3. **Compute shapelet distance features.** For each input window `X` of shape `(T, 2)` and each shapelet `S_k` of shape `(L_k, 2)`, compute the minimum Euclidean distance by sliding `S_k` across all valid positions in `X`:
   ```python
   d_k = min_{t} || X[t:t+L_k, :] - S_k ||_F
   ```
   Stack all `K` distances into a shapelet feature vector `f_shape` of length `K`.

4. **Prepare LLM input tokens.** Segment the same I/Q window into patches (e.g., non-overlapping chunks of 64 time steps). Project each patch through a linear layer to match the LLM's embedding dimension. Prepend a learnable `[CLS]`-style token. Feed the token sequence into the pre-trained LLM with all layers frozen except the final 1–2 transformer blocks and the adapter layers.

5. **Extract LLM features.** Take the output embedding corresponding to the `[CLS]` token (or mean-pool across all output tokens) as the global feature vector `f_llm`.

6. **Fuse shapelet and LLM features.** Concatenate `f_shape` and `f_llm`, then pass through a small MLP projection head to produce the final device embedding `f_fused` of a chosen dimension (e.g., 128 or 256).

7. **Train with cross-entropy loss plus shapelet regularization.** Use standard cross-entropy on the classification logits. Add an L2 regularization term on shapelet parameters to prevent degenerate solutions. Optimize with AdamW; freeze the LLM body and train shapelets, the patch projection, adapter layers, and the classification head.

8. **Evaluate interpretability.** For any classified sample, identify the shapelet with the smallest distance and the temporal position of the best match. Visualize the aligned shapelet overlaid on the original I/Q signal to show what local pattern drove the decision.

9. **Generate prototypes for few-shot cross-domain inference.** Given `M` labeled support samples per device from a new domain, compute each device's prototype as the mean of their `f_fused` vectors. At inference, classify a query by nearest-prototype lookup (cosine similarity or Euclidean distance). No gradient updates are needed.

10. **Deploy and monitor.** Export the trained model (shapelets + LLM + head). At runtime, preprocess incoming I/Q captures identically, compute the fused embedding, and either classify directly or compare against prototypes. Log shapelet match positions to maintain an audit trail of why each authentication decision was made.

## Concrete Examples

**Example 1: Building an IoT Device Authenticator**

User: "I have I/Q signal captures from 20 IoT devices collected in a lab. I need to build a fingerprinting system that can identify which device transmitted a given signal, and I want it to work when we move the devices to a different building."

Approach:
1. Load the dataset — each file contains I/Q samples at shape `(T, 2)` with device labels.
2. Segment into windows of 1024 time steps, split 80/20 for train/test within the lab domain.
3. Initialize 16 shapelets with lengths [32, 64, 96, 128, 160, 192, 224, 256] (2 per length).
4. Set up a GPT-2-small backbone with all layers frozen except the last transformer block.
5. Train for 50 epochs with AdamW (lr=1e-4 for shapelets/head, lr=1e-5 for unfrozen LLM layers).
6. Evaluate on lab test set (expect >95% accuracy).
7. For the new building: collect 5 samples per device, compute prototypes, run few-shot evaluation.

Output:
```
Lab domain accuracy:  97.3%  (standard classification)
New building (5-shot): 91.8%  (prototype-based, no retraining)
Top discriminative shapelet: S_7 (length=128), matches turn-on transient at t=44–172
```

**Example 2: Explaining a Classification Decision**

User: "Our RF fingerprinting system flagged device #12 as an impostor. Can you show why?"

Approach:
1. Feed the flagged I/Q capture through the trained pipeline.
2. Compute shapelet distances — find the best-matching position for each shapelet.
3. Compare the shapelet distance profile against the enrolled prototype for device #12.
4. Identify which shapelets have anomalously large distances (indicating mismatch).

Output:
```
Device #12 prototype distances vs. query signal:

Shapelet  | Length | Prototype Dist | Query Dist | Deviation
S_3       | 64     | 0.12           | 0.87       | +6.25x   *** anomaly
S_11      | 192    | 0.31           | 1.44       | +4.65x   *** anomaly
S_7       | 128    | 0.08           | 0.11       | +1.37x   (normal)

Interpretation: The query signal lacks the characteristic short transient
pattern (S_3, 64 steps) and the oscillator drift pattern (S_11, 192 steps)
that real device #12 exhibits. Likely a different hardware transmitter.
```

**Example 3: Implementing the 2D Shapelet Distance in PyTorch**

User: "Show me how to implement the 2D shapelet distance computation efficiently in PyTorch."

Output:
```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class ShapeletLayer(nn.Module):
    def __init__(self, num_shapelets, shapelet_lengths, n_channels=2):
        """
        num_shapelets: total number of shapelets
        shapelet_lengths: list of int, one per shapelet
        n_channels: 2 for I/Q
        """
        super().__init__()
        self.shapelets = nn.ParameterList([
            nn.Parameter(torch.randn(length, n_channels) * 0.01)
            for length in shapelet_lengths
        ])

    def forward(self, x):
        """
        x: (batch, T, 2) — I/Q signal windows
        returns: (batch, num_shapelets) — min distances
        """
        batch, T, C = x.shape
        distances = []
        for s in self.shapelets:
            L = s.shape[0]
            # Unfold x into sliding windows: (batch, num_windows, L, C)
            windows = x.unfold(1, L, 1)          # (batch, num_windows, C, L)
            windows = windows.permute(0, 1, 3, 2) # (batch, num_windows, L, C)
            # Frobenius distance to shapelet
            diff = windows - s.unsqueeze(0).unsqueeze(0)  # broadcast
            dist = torch.sqrt((diff ** 2).sum(dim=(-1, -2)) + 1e-8)
            min_dist, _ = dist.min(dim=1)  # (batch,)
            distances.append(min_dist)
        return torch.stack(distances, dim=1)  # (batch, K)
```

## Best Practices

- **Do:** Use multiple shapelet lengths spanning at least a 4x range (e.g., 32–256) to capture both transient and sustained hardware signatures.
- **Do:** Freeze most LLM layers and only fine-tune the last 1–2 blocks plus adapters — this preserves generalization and keeps training fast.
- **Do:** Normalize I/Q channels per-sample before shapelet computation; unnormalized signals will cause shapelets to learn amplitude rather than shape.
- **Do:** Visualize shapelet matches on misclassified samples during development — this is the main interpretability advantage; use it for debugging.
- **Avoid:** Training shapelets and the full LLM end-to-end from scratch. The LLM body should stay frozen or nearly frozen; otherwise, you lose the generalization benefit and overfit to the source domain.
- **Avoid:** Using only a single shapelet length. Uniform-length shapelets miss either short transients or longer oscillator patterns, degrading accuracy by 5–15% in cross-domain settings.

## Error Handling

- **Shapelet collapse:** If multiple shapelets converge to the same pattern, add a diversity regularization term (e.g., maximize pairwise distance between shapelets) or re-initialize duplicates mid-training.
- **LLM memory overflow on long signals:** Segment signals into shorter windows before tokenization. If GPU memory is still tight, reduce the patch size or use a smaller LLM (GPT-2 small at 124M parameters is sufficient for RF signals).
- **Poor cross-domain few-shot accuracy:** Ensure prototypes are computed from signals preprocessed identically to training data. Domain-specific normalization differences are the most common cause of degraded prototype quality.
- **Shapelet distances all near zero:** The shapelets are too short or the signal is over-smoothed. Increase minimum shapelet length or reduce preprocessing filtering.

## Limitations

- Requires raw I/Q signal access — does not work with demodulated/decoded packet data or RSSI-only measurements.
- LLM backbone adds inference latency (10–50ms per sample on GPU); for sub-millisecond authentication requirements, use the shapelet branch alone as a lightweight fallback.
- Few-shot prototype accuracy degrades below ~85% when support sets drop to 1-shot in high-noise environments; 3–5 shots are recommended for production use.
- The framework assumes device hardware imperfections are stable over time; devices whose RF characteristics drift (e.g., due to temperature or aging) require periodic prototype refresh.
- Interpretability is limited to shapelet-level explanations; the LLM branch remains a black box contributing to the fused representation.

## Reference

Zhao, T., Zhang, J., Xu, H., Sun, X., & Dai, J. (2026). *Generalizable and Interpretable RF Fingerprinting with Shapelet-Enhanced Large Language Models.* arXiv:2602.03035v1. https://arxiv.org/abs/2602.03035v1

Key sections to consult: Section III (2D shapelet formulation and LLM fusion architecture), Section IV (prototype-based few-shot inference), and Section V (cross-domain experimental results across six datasets).