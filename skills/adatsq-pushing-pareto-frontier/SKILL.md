---
name: "adatsq-pushing-pareto-frontier"
description: >
  Apply AdaTSQ temporal-sensitivity quantization to compress Diffusion Transformers (DiTs) for
  edge deployment. Implements Pareto-aware timestep-dynamic bit-width allocation via beam search
  and Fisher-guided temporal calibration for post-training quantization of models like Flux, Wan,
  and other DiT architectures.
  Trigger phrases:
  - "quantize a diffusion transformer"
  - "compress Flux model for edge deployment"
  - "mixed-precision quantization for DiT"
  - "reduce diffusion model memory footprint"
  - "post-training quantization for image/video generation"
  - "optimize DiT inference with low-bit quantization"
---

# AdaTSQ: Temporal-Sensitivity Quantization for Diffusion Transformers

This skill enables Claude to apply the AdaTSQ post-training quantization (PTQ) framework to compress Diffusion Transformer models (Flux-Dev, Flux-Schnell, Z-Image, Wan2.1, and similar architectures) for efficient deployment. The core technique exploits the fact that different diffusion timesteps have wildly different sensitivity to quantization noise, so uniform bit-width assignment wastes precision on insensitive timesteps while starving critical ones. AdaTSQ uses beam search over a Pareto frontier to dynamically assign per-layer bit-widths at each timestep, and Fisher-information-guided calibration to focus weight optimization on the timesteps that matter most.

## When to Use

- When a user wants to deploy a Diffusion Transformer (Flux, Wan, SDXL-DiT, etc.) on edge or memory-constrained hardware
- When quantizing a DiT model and standard PTQ methods (GPTQ, AWQ, SVDQuant) produce visible quality degradation
- When the user needs mixed-precision quantization that varies across diffusion timesteps, not just across layers
- When building an inference pipeline that must hit a specific memory or compute budget while maximizing generation quality
- When comparing or benchmarking quantization strategies for video generation models like Wan2.1
- When implementing a custom quantization search over bit-width configurations for any iterative generative model

## Key Technique

**Why uniform quantization fails for DiTs:** In diffusion models, early timesteps (high noise) and late timesteps (fine detail refinement) impose fundamentally different demands on model precision. Attention layers at detail-critical timesteps are highly sensitive to quantization, while feedforward layers at noisy timesteps tolerate aggressive compression. Applying a flat W4A8 policy everywhere either wastes bits on insensitive layers or destroys quality at sensitive ones.

**Pareto-aware timestep-dynamic bit-width allocation:** AdaTSQ models the quantization policy search as a constrained pathfinding problem. At each diffusion timestep t, a set of candidate bit-width configurations is generated (informed by Fisher sensitivity of each layer at that timestep). A beam search maintains M non-dominated paths on the Pareto frontier of (cumulative reconstruction error, cumulative bit budget). At each step, every retained path is expanded with every candidate configuration, producing M x |C_t| new paths; only the Pareto-optimal subset is kept. After all T timesteps, surviving paths undergo lightweight end-to-end generation evaluation (e.g., CLIP score), and the best quality-per-bit path is selected. On a single A100, the search for a 50-step Flux-Dev model takes roughly 4 minutes.

**Fisher-guided temporal calibration:** To optimize quantized weights, AdaTSQ computes per-layer, per-timestep Fisher information (expected squared gradient of loss w.r.t. weights). These scores are converted to temporal importance weights via temperature-scaled softmax: alpha_{t,l} = exp(I_{t,l}/tau) / sum(exp(I_{t',l}/tau)). The calibration objective is then a temporally-weighted Hessian: H'_l = sum(alpha_{t,l} * X_{t,l} * X_{t,l}^T). In practice, this is implemented by scaling input activations by sqrt(alpha_{t,l}) before feeding them into a standard GPTQ-style solver, requiring zero changes to the solver itself.

## Step-by-Step Workflow

1. **Profile the DiT model's temporal sensitivity.** Run a small calibration set (e.g., 128-512 prompts) through the full diffusion pipeline at FP16. At each timestep, record per-layer activation statistics (mean, variance) and compute Fisher information scores I_{t,l} = E[(dL/dW_{t,l})^2] for every layer l at every timestep t.

2. **Cluster layers into sensitivity tiers at each timestep.** Using the Fisher scores, group layers into 3-4 tiers (e.g., high/medium/low/negligible sensitivity). At each timestep, assign candidate bit-width sets: high-sensitivity layers get {8, 6, 4}-bit candidates; low-sensitivity layers get {4, 3, 2}-bit candidates. This forms the candidate set C_t for each timestep.

3. **Define the bit-budget constraint.** Set the target average bit-width (e.g., 3.1 bits effective for aggressive compression, 4.0 bits for quality-preserving). This becomes the constraint in the beam search: sum of per-layer bits across all timesteps must not exceed T x L x target_avg.

4. **Run Pareto-aware beam search.** Initialize M paths (M=16-64 is typical) at t=1 with diverse configurations from C_1. For each subsequent timestep, expand all paths with all candidates, compute per-path (cumulative MSE loss, cumulative bits), and prune to the Pareto frontier. Retain at most M paths after pruning.

5. **Evaluate surviving paths end-to-end.** Generate a small batch of images/videos (8-16 samples) using each surviving quantization policy. Score with task-appropriate metrics (CLIP score for text-to-image, FVD for video). Select the policy that maximizes quality within the bit budget.

6. **Compute temporal importance weights for calibration.** From the Fisher scores in step 1, compute alpha_{t,l} = softmax(I_{t,l} / tau) across timesteps for each layer. Set tau to prevent distribution collapse (start with tau = mean(I) and tune if needed).

7. **Run Fisher-weighted GPTQ calibration.** For each layer, collect calibration activations from all timesteps. Scale each activation matrix X_{t,l} by sqrt(alpha_{t,l}). Accumulate the Hessian H'_l = sum(alpha_{t,l} * X_{t,l} * X_{t,l}^T) and run standard GPTQ or similar block-wise quantization solver.

8. **Pack the quantized model with per-timestep dispatch tables.** Store the bit-width configuration as a mapping: (timestep, layer) -> bit_width. At inference time, the dispatch table selects the correct quantized kernel for each layer at each timestep.

9. **Validate with perceptual metrics.** Generate 1K+ samples and measure FID, CLIP score, LPIPS, or GenEval. Compare against FP16 baseline and uniform quantization. Expect <1% quality loss at W4A8 equivalent and viable generation at W3A3 equivalent.

10. **Export for edge deployment.** Convert the quantized model to the target runtime format (ONNX, TensorRT, or custom CUDA kernels). Ensure the inference engine supports mixed-precision dispatch per timestep.

## Concrete Examples

**Example 1: Quantizing Flux-Dev for GPU-constrained deployment**

User: "I need to run Flux-Dev on a single RTX 4060 (8GB VRAM). The FP16 model is 24GB. How do I quantize it without destroying quality?"

Approach:
1. Profile Flux-Dev across its 50 diffusion steps with 256 calibration prompts from COCO captions
2. Compute Fisher information per layer per timestep -- expect attention Q/K projections at steps 35-50 (detail refinement) to show 10-100x higher sensitivity than FFN layers at steps 1-10
3. Set target average bit-width to 3.1 bits (achieves ~6x compression: 24GB -> ~4GB)
4. Run beam search with M=32 paths. The search will converge to approximately 80% W3, 10% W4, 10% W8 distribution, concentrating 8-bit precision on attention layers at late timesteps
5. Calibrate with Fisher-weighted GPTQ -- scale activations from steps 35-50 by ~3x compared to steps 1-10
6. Validate: generate 1K images and measure CLIP score. Expect <0.5% degradation vs FP16

Output:
```
Quantization policy summary for Flux-Dev (50 steps):
  Steps  1-15: All layers W3A8 (low sensitivity, aggressive compression)
  Steps 16-35: Attention W4A8, FFN W3A8 (moderate sensitivity)
  Steps 36-50: Attention Q/K W8A8, Attention V/O W4A8, FFN W3A8 (high sensitivity)

  Effective average: 3.1 bits/weight
  Model size: 4.1 GB (down from 24 GB)
  CLIP score: 0.312 (FP16: 0.314)
  Peak VRAM at inference: 5.8 GB (fits RTX 4060)
```

**Example 2: Implementing Fisher-weighted calibration for a custom DiT**

User: "I have a custom DiT with 28 transformer blocks and 20 diffusion steps. I want to add temporal-aware calibration to my existing GPTQ pipeline."

Approach:
1. Instrument the model to capture gradients during calibration forward passes
2. Compute Fisher scores per layer per timestep
3. Integrate temporal weighting into the existing GPTQ Hessian accumulation

Output:
```python
import torch

def compute_fisher_scores(model, calibration_loader, num_timesteps=20):
    """Compute per-layer, per-timestep Fisher information."""
    fisher = {}  # fisher[(t, layer_name)] = scalar score

    for batch in calibration_loader:
        for t in range(num_timesteps):
            model.zero_grad()
            out_fp = model.forward_at_timestep(batch, t)
            out_q = model.forward_at_timestep_quantized(batch, t)
            loss = torch.nn.functional.mse_loss(out_q, out_fp.detach())
            loss.backward()

            for name, param in model.named_parameters():
                score = (param.grad ** 2).sum().item()
                key = (t, name)
                fisher[key] = fisher.get(key, 0.0) + score

    # Normalize per layer
    for key in fisher:
        fisher[key] /= len(calibration_loader)
    return fisher


def compute_temporal_weights(fisher, num_timesteps, layer_name, tau=None):
    """Convert Fisher scores to softmax importance weights."""
    scores = torch.tensor([fisher[(t, layer_name)] for t in range(num_timesteps)])
    if tau is None:
        tau = scores.mean().item()  # reasonable default
    weights = torch.softmax(scores / tau, dim=0)
    return weights  # shape: (num_timesteps,)


def accumulate_fisher_weighted_hessian(activations_by_timestep, weights):
    """Build temporally-weighted Hessian for GPTQ.

    activations_by_timestep: dict[int, Tensor] of shape (n_samples, in_features)
    weights: Tensor of shape (num_timesteps,)
    """
    H = None
    for t, X in activations_by_timestep.items():
        # Scale activations by sqrt(alpha_t) so that X^T X becomes alpha_t * X^T X
        X_scaled = X * (weights[t].sqrt())
        H_t = X_scaled.T @ X_scaled
        H = H_t if H is None else H + H_t
    return H  # Feed this into standard GPTQ solver
```

**Example 3: Beam search for mixed-precision policy on Wan2.1 video model**

User: "I want to find the best mixed-precision quantization policy for Wan2.1-1.3B video generation within a 4-bit average budget."

Approach:
1. Profile Wan2.1 across its 25 diffusion steps
2. Define candidate configs per sensitivity tier
3. Run Pareto beam search

Output:
```python
def pareto_beam_search(fisher_scores, num_timesteps, num_layers,
                       bit_budget, beam_width=32):
    """Find Pareto-optimal quantization policy via beam search."""

    candidate_bits = {
        'high':   [8, 6, 4],
        'medium': [4, 3],
        'low':    [3, 2],
    }

    def get_candidates(t):
        """Generate per-layer bit configs for timestep t based on Fisher tiers."""
        configs = []
        for layer_idx in range(num_layers):
            tier = classify_sensitivity(fisher_scores, t, layer_idx)
            configs.append(candidate_bits[tier])
        # Return cartesian product (pruned to top-K by greedy heuristic)
        return generate_pruned_configs(configs, max_configs=beam_width)

    # Initialize: list of (cumulative_mse, cumulative_bits, policy_so_far)
    frontier = []
    for config in get_candidates(0):
        mse = evaluate_timestep_mse(model, config, t=0)
        bits = sum(config)
        frontier.append((mse, bits, [config]))

    for t in range(1, num_timesteps):
        candidates_t = get_candidates(t)
        expanded = []
        for (cum_mse, cum_bits, policy) in frontier:
            for config in candidates_t:
                new_mse = cum_mse + evaluate_timestep_mse(model, config, t)
                new_bits = cum_bits + sum(config)
                expanded.append((new_mse, new_bits, policy + [config]))

        # Keep only Pareto-optimal paths
        frontier = pareto_filter(expanded, max_paths=beam_width)

    # Final selection: filter by budget, pick lowest MSE
    valid = [(mse, bits, p) for mse, bits, p in frontier
             if bits <= bit_budget * num_layers * num_timesteps]
    valid.sort(key=lambda x: x[0])
    return valid[0][2]  # Return best policy
```

## Best Practices

- **Do:** Always profile Fisher sensitivity before choosing bit-widths. The sensitivity landscape varies dramatically across architectures -- Flux-Dev attention layers at late timesteps can be 100x more sensitive than early-step FFN layers.
- **Do:** Use temperature scaling (tau) in the softmax for temporal weights. Without it, one or two dominant timesteps suppress all others, collapsing the calibration to a single-timestep optimization.
- **Do:** Start with a beam width of 32 and increase only if the Pareto frontier is too sparse. Larger beams give diminishing returns but linearly increase search cost.
- **Do:** Validate with perceptual metrics (CLIP, FID, LPIPS), not just MSE. Low reconstruction error does not guarantee perceptual quality.
- **Avoid:** Applying a single uniform bit-width across all timesteps. This is the core anti-pattern that AdaTSQ corrects -- even W4A8 uniform fails where W3-mixed-precision with temporal awareness succeeds.
- **Avoid:** Skipping end-to-end evaluation of surviving beam search paths. MSE-guided search is a proxy; final selection must use generation-quality metrics.
- **Avoid:** Using fewer than 128 calibration samples for Fisher estimation. Noisy Fisher scores lead to incorrect sensitivity rankings and wasted precision.

## Error Handling

- **Beam search returns empty frontier:** The bit budget is too tight for the model. Relax the constraint (increase target average bits by 0.5) or reduce the number of candidate configurations to allow coarser solutions.
- **Fisher scores are near-uniform across timesteps:** This can happen with very few diffusion steps (e.g., Flux-Schnell's 4 steps). Fall back to layer-sensitivity-only allocation (skip temporal dimension) and use standard calibration.
- **GPTQ solver diverges with Fisher weighting:** The temperature tau is too low, causing extreme weight concentration. Increase tau or clip alpha weights to a maximum ratio of 10:1 between the most and least weighted timesteps.
- **Generated images show timestep-specific artifacts (e.g., good structure but blurry details):** Late-timestep layers are under-quantized. Increase the minimum bit-width for attention layers at the final 20% of timesteps.
- **OOM during beam search:** Reduce beam width M or evaluate timestep MSE on a smaller calibration batch. The search is embarrassingly parallelizable -- split across GPUs if available.

## Limitations

- **Requires per-timestep dispatch at inference.** The runtime must support loading different quantized kernels per timestep, which adds complexity compared to a single static quantized model. Not all serving frameworks support this natively.
- **Search cost scales with T x L x M.** For models with many timesteps (50+) and many layers (60+), the beam search can become expensive. The 4-minute figure is for Flux-Dev on A100; smaller GPUs or larger models will take proportionally longer.
- **Fisher information is an approximation.** It assumes local quadratic loss landscape, which may not hold for highly nonlinear layers. In practice, this is rarely a problem but can lead to suboptimal allocation for architectures with unusual activation functions.
- **Not applicable to non-iterative models.** The temporal dimension is specific to diffusion (or similar iterative generative) processes. Standard single-pass transformers (LLMs, ViTs) do not benefit from timestep-dynamic allocation.
- **Code not yet publicly released.** As of the paper's publication, the reference implementation at github.com/Qiushao-E/AdaTSQ is pending release. The technique can be implemented from the paper's description using existing GPTQ libraries.

## Reference

- **Paper:** [AdaTSQ: Pushing the Pareto Frontier of Diffusion Transformers via Temporal-Sensitivity Quantization](https://arxiv.org/abs/2602.09883v1) (Zhang et al., 2026). Focus on Section 3 for the beam search formulation and Section 4 for Fisher-weighted calibration integration with GPTQ.