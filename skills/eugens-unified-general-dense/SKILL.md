---
name: "eugens-unified-general-dense"
description: |
  Replace fully-connected feedforward layers with EUGen layers — random-feature-based dense layers that cut inference from O(d^2) to O(md) while preserving expressiveness. Supports drop-in replacement in Transformers, MLPs, and NeRFs, plus backpropagation-free distillation from pretrained weights.
  Trigger phrases:
  - "Replace dense layers with EUGens"
  - "Speed up feedforward layers in my transformer"
  - "Reduce MLP parameter count without losing accuracy"
  - "Distill pretrained weights into efficient layers without backprop"
  - "Approximate dense layers with random features"
  - "Optimize NeRF/MLP inference speed"
---

# EUGens: Efficient, Unified, and General Dense Layers

This skill enables Claude to replace standard fully-connected feedforward layers (FFLs) in neural networks with EUGen layers — a random-feature-based approximation that reduces inference complexity from O(d*l) to O(m*d*k^2 + m*l) where m is much smaller than d or l. EUGens decompose weight-input interactions through random projection matrices and Hadamard products, support arbitrary polynomial activations with unbiased estimation, and incorporate input-norm dependence for strictly greater expressiveness than standard FFLs. The technique applies to any architecture containing dense layers: Transformers, MLPs, NeRFs, and more.

## When to Use

- When a user wants to **speed up inference** of a model bottlenecked by large feedforward layers (e.g., the FFN blocks in a Transformer)
- When a user needs to **reduce parameter count** in dense layers while maintaining model quality
- When a user asks to **compress a pretrained model** without full retraining — EUGens support closed-form layer-wise distillation
- When building **real-time or edge-deployed models** (NeRF rendering, on-device LLMs) where FFL compute is the bottleneck
- When a user wants to **approximate activations** like ReLU, GeLU, or Softplus via polynomial random feature maps
- When replacing MLP layers in **3D reconstruction** pipelines (NeRF, Mip-NeRF 360, iSDF) for faster rendering

## Key Technique

**Random Feature Decomposition.** A standard FFL computes `y = activation(Wx + b)` with weight matrix W of shape (l, d), costing O(d*l). EUGens replace this with a factored form: `EUGen(w, x) = g(w)^T f(x)`, where `f(x)` and `g(w)` are random feature maps built by projecting norm-augmented inputs `x+ = [x; 1; ||x||_2]` through small random matrices G of shape (m, d+2), then taking element-wise (Hadamard) products across polynomial orders up to degree k. The key insight: by choosing m much smaller than d, the layer becomes linear in the input dimension. Polynomial order k (typically 1-2) controls expressiveness; higher k captures higher-order interactions but adds cost proportional to k^2.

**Unbiased Polynomial Activation Approximation.** For any polynomial activation `f(x) = sum(a_i * x^i, i=0..k)`, Theorem 3.1 proves that appropriately initialized random matrices yield `E[EUGen(W, x)] = f(Wx)` — an unbiased estimate. This extends to approximating ReLU/GeLU via low-degree polynomial fits. The variance decreases as O(1/m), and concentration bounds guarantee exponentially decreasing error probability with m.

**Backpropagation-Free Distillation.** For pretrained models, EUGens support an analytic layer-wise knowledge transfer: capture input-output pairs from the original FFL, then solve for optimal EUGen parameters via closed-form least-squares (MSE minimization) without any gradient computation. This enables drop-in replacement of individual layers in already-trained models with 22-27% inference speedup and negligible quality loss.

## Step-by-Step Workflow

1. **Identify target layers.** Profile the model to find FFL bottlenecks. In Transformers, these are the two-layer FFN blocks in each attention layer. In NeRFs, these are the MLP density/color networks. Measure baseline latency and parameter counts.

2. **Choose hyperparameters m and k.** Set polynomial order `k` (start with k=2 for most tasks; k=1 for maximum speed). Set random feature count `m` based on the accuracy-speed tradeoff — typical values are 32, 64, or 128. Rule of thumb: m should be much less than min(d, l) where d is input dim and l is output dim.

3. **Implement the EUGen layer in PyTorch.** Create a module that:
   - Augments input: `x_plus = torch.cat([x, ones, x.norm(dim=-1, keepdim=True)], dim=-1)`
   - Applies random projections: `proj_i = G_i @ x_plus.T` for each order i
   - Computes Hadamard products across orders: `features = prod(proj_1, ..., proj_i)` for i in 0..k
   - Concatenates features across orders and applies output projection

4. **Initialize random matrices G.** Sample entries from a zero-mean distribution (Gaussian or Rademacher) scaled per Theorem 3.1: `G_ij ~ 1/(sigma * m^(1/2i) * |a_i|^(1/2i)) * D_ij`. For variance reduction, use Gaussian Orthogonal Matrices (GOMs) — take the first m rows of a random d-dimensional orthogonal matrix. Set last column to zero if you want standard FFL approximation (no norm dependence).

5. **Make G matrices trainable.** After initialization, register G matrices as `nn.Parameter` so they can be fine-tuned during training. This consistently improves quality over fixed random projections.

6. **For pretrained models: apply analytic distillation.** For each target layer, run a calibration pass to collect (input, output) pairs. With G fixed, solve the least-squares problem `min ||EUGen_output - target_output||^2` in closed form to find optimal output projection weights. No backpropagation needed.

7. **Replace layers selectively.** Start by replacing 1-3 layers and benchmark. Not all layers benefit equally — early layers in Transformers are often more sensitive. Replace layers from the end (deepest) first, as these tend to be more compressible.

8. **Validate quality.** Compare outputs of the original and EUGen-replaced layers on a held-out calibration set. Check that per-layer MSE is within acceptable bounds (< 1% relative error on activations). Run end-to-end evaluation on the target task.

9. **Benchmark inference.** Measure wall-clock latency, peak memory, and throughput. Expected gains: 20-27% inference speedup, up to 30% memory reduction for layers where m << d.

10. **Combine with orthogonal optimizations.** EUGens are complementary to quantization, pruning, and sparse attention. Apply them together for compounding efficiency gains.

## Concrete Examples

**Example 1: Speeding up a GPT-2 FFN block**

User: "The feedforward layers in my GPT-2 model are too slow for real-time inference. Can you replace them with something faster?"

Approach:
1. Identify the FFN blocks: each has two Linear layers (768 -> 3072, then 3072 -> 768) with GeLU activation
2. Replace the first Linear + GeLU with a k=2 EUGen layer using m=64 random features
3. The second Linear (projection back to 768) stays as-is or gets its own EUGen replacement

Output (PyTorch):
```python
import torch
import torch.nn as nn
import math

class EUGenLayer(nn.Module):
    """EUGen dense layer replacing Linear + activation."""
    def __init__(self, in_features, out_features, m=64, k=2):
        super().__init__()
        self.in_features = in_features
        self.out_features = out_features
        self.m = m
        self.k = k
        d_aug = in_features + 2  # input + bias + norm

        # Random projection matrices for each order (trainable)
        self.G = nn.ParameterList([
            nn.Parameter(torch.randn(m, d_aug) / math.sqrt(m))
            for _ in range(k)
        ])
        # Output projection: maps concatenated features to output
        total_features = m * (k + 1)  # order 0 through k
        self.W_out = nn.Linear(total_features, out_features, bias=True)

    def _augment_input(self, x):
        """Append 1 and ||x||_2 to input."""
        ones = torch.ones(*x.shape[:-1], 1, device=x.device, dtype=x.dtype)
        norms = x.norm(dim=-1, keepdim=True)
        return torch.cat([x, ones, norms], dim=-1)

    def forward(self, x):
        x_aug = self._augment_input(x)          # (..., d+2)
        # Order 0: constant features (just ones scaled)
        features = [torch.ones(*x.shape[:-1], self.m, device=x.device, dtype=x.dtype)]
        # Orders 1..k: cumulative Hadamard products of projections
        running = None
        for i in range(self.k):
            proj = x_aug @ self.G[i].T           # (..., m)
            running = proj if running is None else running * proj
            features.append(running)
        features = torch.cat(features, dim=-1)   # (..., m*(k+1))
        return self.W_out(features)


# Drop-in replacement for GPT-2 FFN
class EUGenFFN(nn.Module):
    def __init__(self, d_model=768, d_ff=3072, m=64, k=2):
        super().__init__()
        self.up = EUGenLayer(d_model, d_ff, m=m, k=k)  # replaces Linear+GeLU
        self.down = nn.Linear(d_ff, d_model)

    def forward(self, x):
        return self.down(self.up(x))
```

**Example 2: Analytic distillation of a pretrained NeRF MLP**

User: "I have a trained NeRF model and need faster rendering. Can I compress the MLP layers without retraining?"

Approach:
1. Collect calibration data by running inference on the training set, storing inputs and outputs for each MLP layer
2. For each target layer, construct EUGen random features from inputs
3. Solve the closed-form least-squares for the output projection
4. Replace the original layer and validate rendering quality

Output:
```python
import torch

def distill_layer_analytic(original_layer, calibration_inputs, m=64, k=2):
    """Replace a pretrained Linear layer with EUGen via closed-form solution."""
    device = calibration_inputs.device

    # Collect target outputs from original layer
    with torch.no_grad():
        target_outputs = original_layer(calibration_inputs)  # (N, out_features)

    # Build EUGen and compute random features on calibration data
    eugen = EUGenLayer(
        original_layer.in_features,
        original_layer.out_features,
        m=m, k=k
    ).to(device)

    # Freeze random projections for analytic solve
    with torch.no_grad():
        x_aug = eugen._augment_input(calibration_inputs)
        features_list = [torch.ones(calibration_inputs.shape[0], m, device=device)]
        running = None
        for i in range(k):
            proj = x_aug @ eugen.G[i].T
            running = proj if running is None else running * proj
            features_list.append(running)
        F = torch.cat(features_list, dim=-1)  # (N, m*(k+1))

    # Closed-form least-squares: W_out = (F^T F)^{-1} F^T Y
    FtF = F.T @ F + 1e-5 * torch.eye(F.shape[1], device=device)  # regularize
    FtY = F.T @ target_outputs
    W_optimal = torch.linalg.solve(FtF, FtY)

    # Load solution into EUGen output projection
    with torch.no_grad():
        eugen.W_out.weight.copy_(W_optimal.T)
        eugen.W_out.bias.zero_()

    return eugen

# Usage: replace NeRF MLP layers one at a time
# for layer_idx in [5, 4, 3]:  # start from deepest
#     model.layers[layer_idx] = distill_layer_analytic(
#         model.layers[layer_idx], calibration_data, m=64, k=2
#     )
```

**Example 3: Reducing ViT parameter count**

User: "My ViT-Base has 86M parameters. I want to shrink the FFN blocks to fit on a smaller device."

Approach:
1. ViT-Base FFN: 768 -> 3072 -> 768 per block, 12 blocks. FFN params: ~28M per direction * 12 = ~56M total
2. Replace with EUGen (m=64, k=2): each EUGen layer uses m*(k+1) = 192 feature dims mapped to output, reducing from 768*3072 = 2.4M params to 64*770*2 + 192*3072 ~= 690K params per layer
3. Fine-tune on ImageNet for a few epochs to recover accuracy

Output:
```python
def replace_vit_ffn_blocks(model, m=64, k=2, layers_to_replace=None):
    """Replace FFN blocks in a Vision Transformer with EUGen layers."""
    if layers_to_replace is None:
        layers_to_replace = list(range(len(model.blocks)))

    for idx in layers_to_replace:
        block = model.blocks[idx]
        d_model = block.mlp.fc1.in_features
        d_ff = block.mlp.fc1.out_features
        block.mlp = EUGenFFN(d_model=d_model, d_ff=d_ff, m=m, k=k)

    orig_params = sum(p.numel() for p in model.parameters())
    print(f"New parameter count: {orig_params:,}")
    return model
```

## Best Practices

- **Do:** Start with k=2 and m=64 as defaults. These values work well across Transformers, MLPs, and NeRFs in the paper's experiments.
- **Do:** Use Gaussian Orthogonal Matrices (GOMs) instead of plain Gaussian for the random projections — they reduce variance without extra compute cost. Generate by QR-decomposing a random Gaussian matrix and taking the first m rows.
- **Do:** Make the G matrices trainable after initialization. Fixed random projections work for distillation but trainable ones consistently outperform when training end-to-end.
- **Do:** Replace layers incrementally and validate after each replacement. Deeper layers are generally more compressible than early ones.
- **Avoid:** Setting m too close to d — this eliminates the computational advantage. The whole point is m << min(d, l).
- **Avoid:** Using k > 3 in practice. The paper finds k <= 2 sufficient for all tested tasks, and higher k increases cost proportionally to k^2 with diminishing returns.
- **Avoid:** Replacing all layers simultaneously without calibration. The analytic distillation approach requires per-layer calibration data and should be applied sequentially.

## Error Handling

- **High approximation error after distillation:** Increase m (e.g., from 64 to 128) or increase k from 1 to 2. Also check that the calibration dataset is representative — use at least 1000-5000 diverse samples.
- **NaN or instability during training:** The random feature products can amplify values for large k. Add layer normalization after the feature concatenation, or reduce the initialization scale of G matrices.
- **No speedup observed:** The benefit materializes when m << d. If the input dimension is already small (e.g., d < 128), EUGens may not help. Also verify that the bottleneck is actually in the FFLs, not in attention or data loading.
- **Memory issues during distillation:** The closed-form solve requires computing F^T F of shape (m*(k+1), m*(k+1)). For large m, solve in mini-batches using running sums of F^T F and F^T Y.
- **Accuracy degradation in early Transformer layers:** Early layers tend to be more sensitive. Try keeping the first 1-3 layers as standard FFLs and only replacing later layers.

## Limitations

- EUGens approximate FFLs via random features — they introduce stochastic noise that may be unacceptable for tasks requiring exact numerical precision (e.g., scientific computing).
- The technique works best when the feedforward layer dimensions (d, l) are large. For small networks (d < 128), the overhead of augmentation and feature construction can negate the savings.
- Polynomial activation approximation introduces bias for non-polynomial activations (ReLU, GeLU). The approximation quality depends on how well a low-degree polynomial fits the target activation in the input's operating range.
- The analytic distillation produces a good starting point but may need a short fine-tuning phase (a few hundred steps) for quality-critical applications.
- Current results demonstrate up to 27% speedup — this is meaningful but not transformative. EUGens are best combined with other compression techniques (quantization, pruning) for maximum impact.

## Reference

**Paper:** [EUGens: Efficient, Unified, and General Dense Layers](https://arxiv.org/abs/2601.22563) — Kim, Kim, Sehanobish, Basu Roy Chowdhury, Kidambi (2026). See also predecessor: [arXiv:2410.09771](https://arxiv.org/abs/2410.09771). Focus on Section 3 (Theorems 3.1-3.3) for the mathematical foundation, Section 4 for the distillation algorithm, and Section 5 for experimental configurations and hyperparameter ablations.