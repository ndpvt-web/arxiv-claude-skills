---
name: "hqp-sensitivity-aware-hybrid-quantization"
description: "Apply the HQP framework to compress and accelerate PyTorch models for edge deployment using sensitivity-aware structural pruning followed by 8-bit post-training quantization. Trigger phrases: 'optimize model for edge', 'prune and quantize model', 'compress model for Jetson', 'reduce inference latency on edge device', 'hybrid quantization and pruning', 'deploy model to edge with size constraints'"
---

# HQP: Sensitivity-Aware Hybrid Quantization and Pruning for Edge AI

This skill enables Claude to implement the HQP (Hybrid Quantization and Pruning) framework from scratch in PyTorch. HQP combines Fisher-Information-guided structural pruning with 8-bit post-training quantization in a strict sequential pipeline — prune first under an accuracy constraint, then quantize the already-sparse model. The key insight is that sensitivity-aware pruning produces sparse architectures that are inherently more robust to subsequent quantization noise, yielding up to 3.12x inference speedup and 55% size reduction on NVIDIA Jetson platforms while keeping accuracy loss below 1.5%.

## When to Use

- When the user asks to compress a CNN (MobileNet, ResNet, EfficientNet, etc.) for deployment on edge hardware like NVIDIA Jetson, Raspberry Pi, or mobile devices
- When the user wants to combine pruning and quantization into a single pipeline instead of applying them independently
- When the user needs to meet a strict latency or model-size budget while guaranteeing a maximum accuracy drop
- When the user asks to export a PyTorch model to TensorRT, ONNX Runtime, or another edge inference engine with INT8 support
- When the user is comparing compression strategies and wants a principled approach that outperforms magnitude pruning or naive quantization alone
- When the user mentions Fisher Information, sensitivity-based pruning, or structured filter pruning in the context of model optimization

## Key Technique

**Sensitivity-aware structural pruning.** Standard magnitude-based pruning removes filters with the smallest L1/L2 norms, but small weights can still be critical to accuracy. HQP instead approximates the diagonal of the Fisher Information Matrix (FIM) — the expected squared gradient of the loss with respect to each parameter — using a small calibration set. For a filter `f` in layer `l`, the sensitivity score is `S(f) = sum(grad(L, w_f)^2)` accumulated over calibration batches. Filters with the lowest sensitivity scores contribute least to the loss surface and are safest to remove. This is a structural (not unstructured) operation: entire convolutional filters are zeroed and physically removed, which translates directly to fewer FLOPs without needing sparse-tensor hardware support.

**Strict prune-then-quantize sequencing.** HQP enforces a hard gate between the two stages. Pruning iterates — removing a small fraction of filters per round, fine-tuning briefly, and validating against a user-specified maximum accuracy drop `delta_max` (typically 1.0–1.5%). Only after pruning converges within the accuracy budget does the pipeline proceed to 8-bit post-training quantization (PTQ). Because the pruned model already has a simplified weight distribution (fewer filters, re-distributed magnitudes from fine-tuning), the quantization step introduces less error than it would on the original dense model. The quantized model is then exported for hardware-accelerated INT8 inference.

**Hardware-agnostic deployment.** The pruned-and-quantized model is a standard dense INT8 graph (no sparse tensor tricks needed). It runs on any platform supporting INT8 convolutions: TensorRT on Jetson, ONNX Runtime on ARM, Core ML on Apple Silicon, or TFLite on Android.

## Step-by-Step Workflow

1. **Load the pretrained model and a representative calibration dataset.** Use 500–2000 samples from the training distribution. Wrap the dataset in a DataLoader with batch size 32. Record the baseline accuracy on the validation set — this is the anchor for `delta_max`.

2. **Compute per-filter Fisher sensitivity scores.** For each calibration batch, run a forward pass, compute the cross-entropy loss, and backpropagate. For every convolutional filter `f`, accumulate `S(f) += sum(w_f.grad ** 2)`. After all calibration batches, normalize scores per layer so that layers with different parameter counts are comparable. Store as a dict mapping `(layer_name, filter_index) -> score`.

3. **Rank filters globally by sensitivity and select candidates for removal.** Sort all filters across the model by ascending sensitivity score. Select the bottom `p%` (start with `p=5` per round) as pruning candidates. Filters in the final classification head or in skip connections that would break residual additions should be excluded or handled by pruning matched pairs.

4. **Physically remove the selected filters.** For each pruned filter in `conv_layer[i]`, remove the corresponding output channel from the weight tensor, the batch-norm parameters (gamma, beta, running_mean, running_var), and the corresponding input channel from the next convolutional layer. Use a dependency graph to propagate shape changes through residual blocks correctly.

5. **Fine-tune the pruned model for 2–5 epochs** on the training set with a reduced learning rate (1/10th of the original). This allows remaining filters to compensate for removed capacity.

6. **Validate accuracy against `delta_max`.** Measure accuracy on the validation set. If `baseline_accuracy - current_accuracy > delta_max`, roll back the last pruning round and stop pruning. If the accuracy drop is within budget, return to step 2 for another pruning round. Repeat until the budget is exhausted or a target FLOPs/parameter reduction is reached.

7. **Apply 8-bit post-training quantization.** Use PyTorch's `torch.quantization.quantize_dynamic` for CPU targets or `torch.ao.quantization` with a calibration pass for static quantization. For TensorRT deployment, export to ONNX and use `trtexec --int8 --calib=calibration_cache`. Choose per-channel symmetric quantization for weights and per-tensor asymmetric for activations — this is the standard configuration for INT8 inference engines.

8. **Benchmark the final model.** Measure inference latency (median of 100 runs after 10 warmup runs), model file size on disk, peak memory usage, and top-1/top-5 accuracy. Compare against the unpruned FP32 baseline and against quantization-only or pruning-only variants to confirm the synergistic benefit.

9. **Export for deployment.** Save the quantized model in the target format: TorchScript for PyTorch Mobile, ONNX for cross-platform, or a TensorRT engine file for Jetson. Include metadata (original accuracy, compressed accuracy, compression ratio, target `delta_max`) in a sidecar JSON for production tracking.

## Concrete Examples

**Example 1: Compress ResNet-18 for Jetson Nano deployment**

User: "I have a ResNet-18 trained on CIFAR-100 with 78.2% accuracy. I need to deploy it on a Jetson Nano with under 10ms inference latency. Compress it as much as possible but don't lose more than 1% accuracy."

Approach:
1. Load the pretrained ResNet-18 and 1000 calibration samples from CIFAR-100 training set
2. Compute Fisher sensitivity scores across all 4 residual groups (8 conv layers with 64→512 filters)
3. Run iterative pruning with `delta_max=1.0`, removing ~5% of filters per round, fine-tuning 3 epochs between rounds
4. After 6 rounds, the model is pruned from 11.2M to ~5.5M parameters with 77.4% accuracy (0.8% drop)
5. Apply static INT8 quantization with 200-sample calibration, yielding 77.2% accuracy
6. Export to ONNX and build TensorRT INT8 engine

Output:
```
HQP Compression Report — ResNet-18 on CIFAR-100
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Baseline accuracy:     78.2%  (FP32, dense)
Post-pruning accuracy: 77.4%  (FP32, 51% filters removed)
Post-quantization acc: 77.2%  (INT8, pruned)
Accuracy drop:         1.0%   (within delta_max=1.0%)

Model size:  44.8 MB → 18.6 MB  (58% reduction)
Parameters:  11.2M  → 5.5M     (51% reduction)
Latency:     28.3ms → 9.1ms    (3.1x speedup, Jetson Nano TensorRT)
```

**Example 2: Optimize MobileNetV3-Small for real-time mobile inference**

User: "My MobileNetV3-Small runs at 15ms on a phone's NPU. I need it under 8ms. Quantization alone gets me to 11ms. Can you do better?"

Approach:
1. Load pretrained MobileNetV3-Small (2.5M params, 67.4% top-1 on ImageNet subset)
2. Compute Fisher scores — MobileNetV3 uses depthwise-separable convolutions, so prune pointwise (1x1) filters only, leaving depthwise layers untouched (they are already cheap)
3. Prune 30% of pointwise filters across squeeze-excitation and expansion layers with `delta_max=1.5%`
4. Fine-tune 5 epochs with cosine LR schedule from 1e-4
5. Apply per-channel INT8 quantization
6. Export to TFLite with INT8 delegate

Output:
```
HQP Compression Report — MobileNetV3-Small
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Baseline accuracy:     67.4%
Post-pruning accuracy: 66.3%  (30% pointwise filters removed)
Post-quantization acc: 66.1%  (INT8)
Accuracy drop:         1.3%

Model size:  10.1 MB → 4.8 MB  (52% reduction)
Latency:     15ms → 6.8ms      (2.2x speedup, mobile NPU)
```

**Example 3: Generating the pruning code from scratch**

User: "Write me the Fisher sensitivity scoring function for structured pruning."

Output:
```python
import torch
import torch.nn as nn
from collections import defaultdict

def compute_fisher_sensitivity(model, dataloader, num_batches=50):
    """Compute per-filter Fisher Information sensitivity scores.

    Returns dict mapping (layer_name, filter_idx) -> sensitivity score.
    Lower scores = safer to prune.
    """
    model.train()
    sensitivity = defaultdict(float)
    criterion = nn.CrossEntropyLoss()

    for batch_idx, (inputs, targets) in enumerate(dataloader):
        if batch_idx >= num_batches:
            break
        inputs, targets = inputs.cuda(), targets.cuda()

        model.zero_grad()
        outputs = model(inputs)
        loss = criterion(outputs, targets)
        loss.backward()

        for name, module in model.named_modules():
            if isinstance(module, nn.Conv2d) and module.weight.grad is not None:
                # Sum of squared gradients per output filter
                grad_sq = module.weight.grad.data ** 2
                # grad_sq shape: [out_channels, in_channels, kH, kW]
                per_filter_score = grad_sq.sum(dim=(1, 2, 3))
                for f_idx in range(per_filter_score.size(0)):
                    sensitivity[(name, f_idx)] += per_filter_score[f_idx].item()

    # Normalize per layer for cross-layer comparability
    layer_names = set(name for name, _ in sensitivity.keys())
    for layer_name in layer_names:
        layer_scores = [
            sensitivity[(layer_name, i)]
            for i in range(max(
                idx for n, idx in sensitivity if n == layer_name
            ) + 1)
        ]
        max_score = max(layer_scores) if layer_scores else 1.0
        for i in range(len(layer_scores)):
            sensitivity[(layer_name, i)] /= max_score + 1e-8

    return dict(sensitivity)
```

## Best Practices

**Do:** Prune iteratively in small increments (3–10% of filters per round) rather than one aggressive cut. Smaller steps let fine-tuning recover accuracy and give the Fisher scores a chance to re-stabilize after each round.

**Do:** Always compute Fisher scores on data from the actual training distribution, not random noise or out-of-distribution samples. The gradient magnitudes are only meaningful relative to the real loss surface.

**Do:** Handle residual/skip connections explicitly. In ResNet-style architectures, if you prune a filter in one branch, the corresponding channel in the skip connection must also be removed to keep tensor dimensions aligned. Build a layer dependency graph before pruning.

**Do:** Benchmark on the actual target hardware, not just on a desktop GPU. INT8 speedups vary dramatically — a Jetson Nano's DLA accelerator behaves differently from its GPU cores, and desktop GPUs often show minimal INT8 benefit for small models.

**Avoid:** Pruning depthwise convolution filters independently. In depthwise-separable architectures (MobileNet family), always prune the pointwise (1x1) layers and propagate the channel removal to the paired depthwise layer. Pruning depthwise filters alone breaks the group structure.

**Avoid:** Skipping the fine-tuning step between pruning rounds. Without even brief re-training (2–5 epochs), accuracy drops accumulate rapidly and the model may fall outside `delta_max` after quantization, wasting the entire pipeline.

**Avoid:** Using dynamic quantization when targeting latency-sensitive edge deployments. Static quantization with proper calibration produces fused, hardware-optimized kernels; dynamic quantization adds per-inference overhead that negates much of the speedup.

## Error Handling

- **Accuracy drops exceed `delta_max` after pruning:** Roll back the last pruning round. Reduce the per-round pruning percentage from 5% to 2% and retry. If accuracy is still too fragile, the model may already be near its compression floor — report the achieved compression ratio and stop.
- **Shape mismatch after filter removal:** The dependency graph is incomplete. Trace the model with a dummy input (`torch.jit.trace`) to discover all layer connections, paying special attention to concatenation ops, squeeze-excitation blocks, and skip connections.
- **Quantization calibration produces NaN or Inf:** Some activation ranges collapse after aggressive pruning. Increase the calibration dataset size (try 500+ samples) and switch from min-max calibration to entropy/percentile calibration (`HistogramObserver` in PyTorch).
- **No latency improvement despite smaller model:** The pruned filter count may not align with hardware-friendly dimensions (multiples of 8 or 32 for CUDA tensor cores). Round filter counts to the nearest hardware-friendly multiple before finalizing the pruned architecture.
- **TensorRT build fails on pruned ONNX model:** Ensure all `Reshape`/`Unsqueeze` ops have static shapes. Dynamic axes from pruning can confuse the TensorRT parser. Pin shapes explicitly during ONNX export with `dynamic_axes=None`.

## Limitations

- **Not designed for transformers or attention-based models.** HQP targets convolutional filter pruning. For Vision Transformers or LLMs, head pruning or block-level sparsity requires different sensitivity metrics (e.g., attention entropy, not FIM diagonal).
- **Fine-tuning requires access to training data.** If you only have a frozen model and no training data, the Fisher scoring step still works (use the calibration set), but without fine-tuning between pruning rounds, accuracy drops will be much steeper.
- **Structured pruning has a coarser granularity floor.** Removing entire filters gives hardware-friendly speedups but can't achieve the extreme compression ratios (10x+) that unstructured or mixed-precision approaches can. HQP typically plateaus around 50–60% parameter reduction for already-efficient models like MobileNet.
- **INT8 quantization benefits are hardware-dependent.** Older edge devices without dedicated INT8 datapaths may see minimal or no latency improvement from the quantization stage. Always profile on actual target hardware.
- **The Fisher approximation assumes a locally quadratic loss surface.** For models far from a local minimum (e.g., partially trained checkpoints), the sensitivity scores may be unreliable. Always start from a converged pretrained model.

## Reference

[HQP: Sensitivity-Aware Hybrid Quantization and Pruning for Ultra-Low-Latency Edge AI Inference](https://arxiv.org/abs/2602.06069v1) — Gopalan & Ali, 2026. Focus on Section 3 (the FIM-based sensitivity metric and conditional pruning algorithm) and Table 1 (per-platform speedup and accuracy results across Jetson devices).