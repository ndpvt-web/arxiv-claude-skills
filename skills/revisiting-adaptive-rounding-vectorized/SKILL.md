---
name: "revisiting-adaptive-rounding-vectorized"
description: "Implement VQRound -- a parameter-efficient adaptive rounding framework for LLM post-training quantization that reparameterizes dense rounding matrices into compact VQ codebooks. Use when: 'quantize a large language model to 4-bit', 'apply adaptive rounding with VQRound', 'reduce PTQ parameters for LLM compression', 'implement codebook-based rounding for quantization', 'optimize rounding decisions for weight quantization', 'integrate VQRound with GPTQ pipeline'."
---

# VQRound: Vectorized Reparameterization for LLM Quantization

This skill enables Claude to implement and guide users through VQRound, a post-training quantization (PTQ) technique that replaces the dense, element-wise rounding matrices used in traditional adaptive rounding with a compact vector quantization (VQ) codebook. VQRound achieves competitive or superior quantization quality to methods like GPTQ and AdaRound while using only 0.07--0.39% of the trainable parameters, making adaptive rounding practical for billion-parameter LLMs. The technique is based on the paper "Revisiting Adaptive Rounding with Vectorized Reparameterization for LLM Quantization" (Zhou et al., 2026).

## When to Use

- When the user wants to quantize an LLM (e.g., LLaMA, OPT, Qwen) to low bit-widths (2-bit, 3-bit, 4-bit) with better accuracy than round-to-nearest (RTN)
- When the user asks about parameter-efficient adaptive rounding or reducing memory cost of rounding optimization
- When implementing a PTQ pipeline and the user wants to integrate codebook-based rounding with existing methods like GPTQ or QuaRot
- When the user is hitting out-of-memory errors with traditional adaptive rounding (AdaRound) on large models
- When the user needs to quantize models at extreme low bit-widths (2-bit or 3-bit) where standard methods collapse
- When the user asks how to initialize rounding matrices using Hessian information for better convergence

## Key Technique

**The core problem:** Standard adaptive rounding (AdaRound) learns a dense rounding matrix with one trainable parameter per weight element -- for a 13B-parameter model, that means 12.69 billion rounding parameters. This is impractical for LLMs.

**VQRound's solution:** Instead of optimizing each rounding value independently, VQRound splits the latent rounding matrix A into L vectors of dimension d, then learns only k codebook centroids (default: k=4096, d=8). Each vector is assigned to its nearest centroid, reducing trainable parameters from O(mn) to O(kd). The codebook values pass through a rectified sigmoid h(x) = clip(sigmoid(x)(zeta - gamma) + gamma, 0, 1) to produce final rounding decisions in [0,1]. A regularizer anneals these soft decisions toward binary 0/1 values during training.

**Why VQ beats low-rank alternatives:** Low-rank methods (LoRA, Kronecker) minimize the Frobenius norm (average error) but cannot control per-element worst-case deviations. LLM weights have heavy-tailed distributions where localized error spikes cause catastrophic accuracy loss. VQ's local blocking structure naturally constrains the L-infinity norm (worst-case element error). The paper proves via Lipschitz analysis that small VQ approximation errors translate to bounded rounding errors, while large errors saturate the clipping function -- meaning VQ's tail suppression is theoretically grounded, not just empirical.

## Step-by-Step Workflow

1. **Collect calibration data.** Sample 128 sequences of length 2048 from C4 (or a domain-appropriate corpus). Pass them through the full-precision model to capture per-layer input activations X and compute the Hessian H = 2X^T X for each linear layer.

2. **Compute standard uniform quantization grids.** For each weight matrix W, determine the quantization scale s and zero-point z using min-max or percentile-based range estimation at the target bit-width (e.g., 4-bit -> 16 levels). Compute the float-grid weights: W_float = clip(W/s + z, 0, 2^b - 1).

3. **Initialize the rounding matrix with Hessian-aware residuals.** Process weight columns sequentially: quantize each column with RTN, compute the residual error, scale by the inverse diagonal Hessian entry, and propagate residuals to unprocessed columns via off-diagonal Hessian terms (Cholesky-based, similar to GPTQ's column-order processing). Store the final residual sign pattern as the initial rounding direction.

4. **Construct the VQ codebook via K-Means.** Reshape the initialized latent rounding matrix A into L vectors of dimension d=8. Run GPU-accelerated K-Means (e.g., FAISS) for 100 iterations to produce k=4096 centroids. Store the codebook C = {c_1, ..., c_k} and assignments {i_1, ..., i_L}.

5. **Optimize the codebook with blockwise reconstruction loss.** For each transformer block, minimize the layer-wise reconstruction objective ||W_q X - W X||^2 by gradient-descending only the codebook centroids. Use the straight-through estimator (STE) to backpropagate through the argmin assignment step. Apply the rounding regularizer R(H) = sum(1 - |2H_ij - 1|^beta) with beta annealed from 20 to 2 over training to encourage binary convergence.

6. **Run end-to-end (E2E) fine-tuning across all layers.** After blockwise convergence, jointly optimize all layer codebooks using KL-divergence distillation loss between the quantized model's output logits and the full-precision model's logits. Train for 5000 steps with Adam (lr=1e-2), 10% warm-up, lambda=1e-2 for the regularizer.

7. **Freeze rounding decisions and extract final quantized weights.** After optimization, convert soft rounding values to hard 0/1 decisions. Apply: W_q = s * (floor(W_float) + round_decision) - z. The codebook is no longer needed at inference -- only the integer weights remain.

8. **Validate perplexity and zero-shot accuracy.** Evaluate on WikiText-2 and C4 for perplexity; run zero-shot benchmarks (WinoGrande, PiQA, HellaSwag, ARC-e, ARC-c) to confirm accuracy retention.

9. **(Optional) Combine with complementary methods.** VQRound stacks with GPTQ (apply GPTQ first, then VQRound on residuals), QuaRot (rotation-aware quantization), or OmniQuant for further gains.

## Concrete Examples

**Example 1: Quantizing LLaMA-7B to 4-bit with VQRound**

User: "I want to quantize LLaMA-7B to 4-bit weights. GPTQ gives me 6.17 perplexity on WikiText-2 but I want better. How do I use VQRound?"

Approach:
1. Load LLaMA-7B in float16. Sample 128 C4 sequences (length 2048) as calibration data.
2. For each linear layer, compute the Hessian from calibration activations.
3. Initialize rounding matrices using the Hessian-aware residual method (step 3 above).
4. Build per-layer VQ codebooks: reshape each rounding matrix into 8-dimensional vectors, run K-Means with k=4096.
5. Optimize codebooks blockwise (reconstruction loss), then run E2E fine-tuning for 5000 steps with KL distillation.
6. Freeze rounding decisions, export integer weights.

Expected output:
```
WikiText-2 perplexity: ~6.13 (vs GPTQ 6.17, RTN ~6.5)
Trainable parameters: ~4.5M (vs 6.7B for dense AdaRound)
GPU memory: fits on a single RTX A6000 (48GB)
```

**Example 2: Extreme 2-bit quantization of LLaMA-7B**

User: "Can I quantize LLaMA-7B to 2-bit? GPTQ completely fails at W2."

Approach:
1. Follow the same pipeline but set bit-width b=2 (4 quantization levels).
2. The Hessian-aware initialization is critical here -- random initialization at 2-bit causes divergence.
3. Increase E2E fine-tuning steps if perplexity has not converged by step 5000.
4. VQRound's L-infinity control prevents the catastrophic collapse seen in GPTQ at 2-bit.

Expected output:
```
WikiText-2 perplexity: ~65.41 (vs GPTQ >100,000 -- effectively collapsed)
Model is degraded but functional for 2-bit; useful for research into extreme compression.
```

**Example 3: Integrating VQRound into an existing GPTQ pipeline**

User: "I already have a GPTQ quantization script. How do I add VQRound on top?"

Approach:
1. Run GPTQ as usual to get initial quantized weights W_q_gptq.
2. Compute the residual rounding error: R = W_float - floor(W_float) - round_decision_gptq.
3. Initialize VQRound's latent matrix from R (the error GPTQ left behind).
4. Build codebook and optimize as in the standard pipeline.
5. The final rounding combines GPTQ's column-reordering with VQRound's codebook optimization.

Expected output:
```
LLaMA2-7B W4 WikiText-2: 5.85 PPL (vs GPTQ-only 6.06)
The two methods are complementary -- GPTQ handles column-wise error propagation,
VQRound handles the remaining per-element rounding optimization.
```

## Implementation Skeleton (PyTorch)

```python
import torch
import faiss

def build_codebook(rounding_matrix, k=4096, d=8, n_iter=100):
    """Reshape rounding matrix into d-dim vectors and cluster with K-Means."""
    flat = rounding_matrix.reshape(-1, d).float().cpu().numpy()
    kmeans = faiss.Kmeans(d, k, niter=n_iter, gpu=True)
    kmeans.train(flat)
    _, assignments = kmeans.index.search(flat, 1)
    centroids = torch.from_numpy(kmeans.centroids).to(rounding_matrix.device)
    assignments = torch.from_numpy(assignments.squeeze()).to(rounding_matrix.device)
    return centroids, assignments

def vq_lookup(centroids, assignments, original_shape, d=8):
    """Reconstruct rounding matrix from codebook."""
    vectors = centroids[assignments]  # (L, d)
    return vectors.reshape(original_shape)

def rectified_sigmoid(a, zeta=1.1, gamma=-0.1):
    """Transform latent values to [0, 1] rounding probabilities."""
    return torch.clamp(torch.sigmoid(a) * (zeta - gamma) + gamma, 0.0, 1.0)

def rounding_regularizer(h, beta):
    """Encourage binary convergence: penalty is zero when h is exactly 0 or 1."""
    return (1.0 - (2 * h - 1).abs().pow(beta)).sum()

# Training loop (simplified):
# for step in range(5000):
#     a_vq = vq_lookup(centroids, assignments, W.shape)
#     h = rectified_sigmoid(a_vq)
#     w_q = scale * (torch.floor(w_float) + h) - zero_point
#     loss = reconstruction_loss(w_q, X, W, X) + lam * rounding_regularizer(h, beta)
#     loss.backward()  # STE through argmin assignment
#     optimizer.step()
#     beta = anneal(beta, step)  # 20 -> 2
```

## Best Practices

- **Do:** Use Hessian-aware initialization (step 3) -- it provides a much better starting point than random or RTN-based initialization and is critical for convergence at low bit-widths.
- **Do:** Anneal the beta parameter from a high value (20) to a low value (2) over training. Starting low causes premature hard decisions; staying high prevents convergence to binary values.
- **Do:** Use FAISS with GPU acceleration for K-Means -- CPU K-Means on millions of vectors is prohibitively slow.
- **Do:** Combine VQRound with GPTQ or QuaRot for best results -- the methods address orthogonal error sources.
- **Avoid:** Using low-rank reparameterizations (LoRA-style) for the rounding matrix. They minimize Frobenius norm and produce localized error spikes that cause NaN values in LLM inference.
- **Avoid:** Skipping the E2E fine-tuning phase. Blockwise optimization alone is a good initialization but the cross-layer distillation loss captures inter-block error accumulation that blockwise cannot.

## Error Handling

| Problem | Cause | Fix |
|---------|-------|-----|
| NaN in perplexity after quantization | Error spikes in rounding (common with low-rank methods) | Verify VQ codebook was properly trained; check that rectified sigmoid clipping is active |
| Codebook collapse (many empty centroids) | Poor K-Means initialization or too few vectors | Reduce k or increase d; ensure rounding matrix is initialized before clustering |
| OOM during Hessian computation | Full Hessian is O(n^2) | Use only diagonal Hessian (H_diag = 2 * (X**2).sum(0)); process columns sequentially |
| Perplexity not converging during E2E | Learning rate too high or beta annealing too fast | Reduce lr to 5e-3; extend annealing schedule; increase warm-up to 20% |
| Quantized model accuracy much worse than expected | Calibration data mismatch with deployment domain | Use domain-representative calibration data; increase from 128 to 256 samples |

## Limitations

- **Inference overhead is zero** (codebook discarded after quantization), but **training requires GPU-accelerated K-Means** which adds a dependency on FAISS or similar libraries.
- At 2-bit quantization, VQRound prevents catastrophic collapse but perplexity is still substantially degraded (~65 PPL for LLaMA-7B). It is not a silver bullet for extreme compression.
- The method is designed for **weight-only quantization**. It does not address activation quantization, which requires separate techniques (e.g., SmoothQuant).
- Codebook hyperparameters (k=4096, d=8) were tuned on LLaMA/OPT families. Other architectures (e.g., Mamba, RWKV) may need re-tuning.
- E2E fine-tuning requires forward passes through the full model, so very large models (70B+) may need model parallelism or gradient checkpointing even though rounding parameters are small.

## Reference

**Paper:** "Revisiting Adaptive Rounding with Vectorized Reparameterization for LLM Quantization" -- Zhou, Chen, Benini, Sun, Li (2026). [arXiv:2602.02151v1](https://arxiv.org/abs/2602.02151v1)

**Code:** [github.com/zhoustan/VQRound](https://github.com/zhoustan/VQRound)

**Key insight to look for:** Theorem 2 and the L-infinity analysis (Section 3.3) -- this is the theoretical justification for why vector quantization controls tail errors better than low-rank alternatives, and it is the core reason VQRound works where other parameter-efficient methods fail.