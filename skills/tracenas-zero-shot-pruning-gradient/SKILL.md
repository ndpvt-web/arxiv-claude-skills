---
name: "tracenas-zero-shot-pruning-gradient"
description: "Implement TraceNAS-style zero-shot LLM structured pruning using gradient trace correlation as a scale-invariant proxy. Jointly prune model depth (layer removal) and width (attention heads, FFN channels) via evolutionary NAS without any training. Trigger phrases: 'prune LLM without training', 'zero-shot model compression', 'structured pruning with NAS', 'find optimal pruned architecture', 'gradient-based pruning proxy', 'compress Llama/Qwen model'"
---

# TraceNAS: Zero-Shot LLM Pruning via Gradient Trace Correlation

This skill enables Claude to guide users through implementing and applying TraceNAS, a training-free Neural Architecture Search framework that discovers optimal structured pruning configurations for Large Language Models. Instead of evaluating layer or head importance in isolation, TraceNAS computes gradient trace correlation between candidate pruned architectures and the original model, capturing global structural dependencies. The result is a scale-invariant zero-shot proxy (Spearman rho = 0.94 with downstream accuracy) that identifies pruned models whose loss landscapes remain aligned with the pretrained model — meaning they recover performance quickly during post-pruning continued pretraining. The entire architecture search runs on a single GPU in under 9 hours, a 10x reduction over training-aware pruning methods.

## When to Use

- When the user wants to compress a large LLM (Llama, Qwen, Mistral, etc.) to a specific parameter budget without expensive retraining-based pruning
- When the user asks how to jointly prune both depth (remove transformer blocks) and width (remove attention heads / FFN channels) non-uniformly
- When the user needs a zero-shot proxy metric to rank pruned model candidates without fine-tuning each one
- When the user wants to implement evolutionary NAS over a structured pruning search space for LLMs
- When the user is building an automated model compression pipeline and needs a fast, reliable importance scoring method
- When the user asks how to evaluate whether a pruned model will recover well during continued pretraining

## Key Technique

**Gradient Trace Correlation as a Zero-Shot Proxy.** TraceNAS computes a single forward-backward pass on a small calibration set (16 sequences, ~65K tokens) for each pruned candidate. It records gradient traces at every surviving layer, then measures the Pearson correlation between the candidate's gradient traces and the base model's gradient traces. This correlation is computed per-layer as `rho(l) = (1/N_l) * <(g_sub(l) - mu)/sigma, (g_base(l) - mu)/sigma>`, where mean-centering and standard deviation normalization make the metric scale-invariant — it captures *directional* alignment independent of magnitude shifts caused by structural removal. The final proxy score aggregates per-layer correlations weighted by parameter retention ratio: `Phi = sum(r_attn(l) * rho(l)) + sum(r_mlp(l) * rho(l))`, preventing aggressively pruned blocks from dominating the score through high-variance noise.

**Joint Depth-Width Search via Evolutionary NAS.** The search space encodes each candidate as a pair `(d, kappa)` where `d` is a binary mask over transformer blocks (depth) and `kappa` specifies per-layer retention ratios for attention and MLP dimensions (width). A population of 30 candidates evolves over 50 iterations using hybrid operators: contiguous-range crossover and bit-flip mutation for depth; interpolation crossover and multiplicative Gaussian jitter for width. Width pruning uses activation-weighted magnitude importance: `I(l,j) = sum_i |W(l,ij)| * ||X(l,j)||_2`, which accounts for outlier features by scaling weight magnitudes by input activation norms. Masks are applied at attention output (W_o) and MLP down projections (W_down), propagated to Q/K/V and up/gate matrices respectively. For GQA models, K/V heads remain unpruned for KV-cache compatibility, and MLP hidden dimensions are rounded to multiples of 32.

**Why It Works.** High proxy score Phi implies the first-order Taylor expansion of the sub-network's loss surface stays congruent with the base model's. Because gradients flow through the full masked graph during backpropagation, upstream bottlenecks (e.g., a removed layer) cause downstream gradient de-correlation — so the metric naturally captures global inter-layer dependencies rather than local saliency.

## Step-by-Step Workflow

1. **Define the target parameter budget.** Determine the desired model size (e.g., Llama-2-7B down to 2.7B parameters). Compute the overall compression ratio and decide minimum depth constraint `L_min` (minimum surviving transformer blocks).

2. **Prepare calibration data.** Sample 16 sequences from a representative corpus (e.g., FineWeb-Edu, RedPajama, or your domain data). Tokenize to the model's context length (4096 for Llama-2/Qwen-2.5, 8192 for Llama-3.1). This yields ~65K calibration tokens.

3. **Compute base model gradient traces.** Run one forward-backward pass on the calibration batch through the full pretrained model. Store per-layer gradient traces `g_base(l)` for all transformer blocks. Use LoRA-style low-rank storage to reduce memory from full gradient tensors.

4. **Compute activation-weighted importance scores.** For each layer, compute `I(l,j) = sum_i |W(l,ij)| * ||X(l,j)||_2` at attention output and MLP down projections. These scores determine which heads/channels to retain at a given width ratio.

5. **Initialize the NAS population.** Generate 30 candidate architectures `(d, kappa)` using global importance priors — rank layers by their base gradient norms and initialize depth masks and width ratios proportionally. Enforce the parameter budget constraint per candidate.

6. **Evaluate each candidate with the gradient trace proxy.** For each candidate: apply the binary depth mask and width masks (via activation-weighted importance ranking), run one forward-backward pass on the calibration batch, record gradient traces `g_sub(l)`, compute per-layer Pearson correlation `rho(l)`, and aggregate into the sparsity-weighted proxy score `Phi`.

7. **Evolve the population over 50 iterations.** Select top candidates by Phi score. Apply depth operators (contiguous-range crossover, bit-flip mutation) and width operators (interpolation crossover, multiplicative Gaussian jitter). Enforce constraints: `sum(d) >= L_min`, width ratios in (0, 1], MLP dimensions rounded to multiples of 32, parameter count within budget.

8. **Extract the optimal architecture.** After 50 generations, select the candidate with the highest Phi score. Record its depth mask and per-layer width configuration as the final pruned architecture specification.

9. **Realize the pruned model.** Physically remove deactivated blocks, prune attention heads and FFN channels per the width mask, and save the resulting model weights. Verify parameter count matches the target budget.

10. **Run continued pretraining for recovery.** Train the pruned model on ~20B tokens (FineWeb-Edu or domain data) with learning rate 1e-4 using a Warmup-Stable-Decay scheduler. This recovers performance to competitive levels with training-aware baselines.

## Concrete Examples

**Example 1: Compress Llama-2-7B to 2.7B parameters**

User: "I want to prune Llama-2-7B down to about 2.7B parameters for edge deployment. Can you help me set up a TraceNAS-style pruning pipeline?"

Approach:
1. Calculate compression ratio: 2.7B / 7B = ~38.6% retention
2. Set up calibration: sample 16 sequences (4096 tokens each) from FineWeb-Edu
3. The search space covers 32 transformer blocks with binary depth mask and per-block width ratios for 32 attention heads and 11008 FFN hidden dim
4. Implement the gradient trace proxy computation:

```python
import torch
from transformers import AutoModelForCausalLM, AutoTokenizer

def compute_gradient_traces(model, calibration_batch):
    """Forward-backward pass to collect per-layer gradient traces."""
    model.zero_grad()
    outputs = model(**calibration_batch, labels=calibration_batch["input_ids"])
    outputs.loss.backward()

    traces = {}
    for name, param in model.named_parameters():
        if param.grad is not None:
            traces[name] = param.grad.detach().clone()
    return traces

def gradient_trace_correlation(traces_sub, traces_base, depth_mask, width_ratios):
    """Compute sparsity-weighted Pearson correlation proxy Phi."""
    phi = 0.0
    for layer_idx, active in enumerate(depth_mask):
        if not active:
            continue
        for block_type in ["self_attn", "mlp"]:
            key_pattern = f"layers.{layer_idx}.{block_type}"
            g_sub = torch.cat([
                traces_sub[k].flatten()
                for k in traces_sub if key_pattern in k
            ])
            g_base = torch.cat([
                traces_base[k].flatten()
                for k in traces_base if key_pattern in k
            ])
            # Standardize (scale-invariant)
            g_sub_norm = (g_sub - g_sub.mean()) / (g_sub.std() + 1e-8)
            g_base_norm = (g_base - g_base.mean()) / (g_base.std() + 1e-8)
            rho = torch.dot(g_sub_norm, g_base_norm) / g_sub_norm.numel()

            # Weight by retention ratio
            r = width_ratios[layer_idx][block_type]
            phi += r * rho.item()
    return phi
```

5. Run evolutionary search (population=30, iterations=50) to find the architecture with highest Phi
6. Extract and save the pruned model

Output: A pruned 2.7B model spec like:
```
Depth: blocks [0,1,2,3,5,6,7,8,10,11,14,15,17,18,20,21,24,25,28,29] retained (20/32)
Width per retained block:
  Block 0:  attn_ratio=0.875, mlp_ratio=0.75
  Block 1:  attn_ratio=1.0,   mlp_ratio=0.625
  Block 3:  attn_ratio=0.75,  mlp_ratio=0.5
  ...
Total parameters: 2.71B
Proxy score Phi: 0.847
```

**Example 2: Implement the activation-weighted importance scoring**

User: "How do I compute importance scores for deciding which attention heads and FFN channels to prune in each layer?"

Approach:
1. Run a calibration forward pass, collecting intermediate activations
2. Compute activation-weighted magnitude importance per output dimension

```python
def compute_importance_scores(model, calibration_batch):
    """Activation-weighted magnitude importance for width pruning."""
    activations = {}

    def hook_fn(name):
        def hook(module, input, output):
            # Store L2 norm of input activations per feature
            activations[name] = input[0].detach().float().norm(dim=(0, 1))  # [hidden_dim]
        return hook

    hooks = []
    for name, module in model.named_modules():
        if "o_proj" in name or "down_proj" in name:
            hooks.append(module.register_forward_hook(hook_fn(name)))

    with torch.no_grad():
        model(**calibration_batch)

    for h in hooks:
        h.remove()

    importance = {}
    for name, module in model.named_modules():
        if name in activations:
            W = module.weight.detach().float().abs()  # [out, in]
            X_norm = activations[name]                # [in]
            # I(l,j) = sum_i |W(l,ij)| * ||X(l,j)||_2
            importance[name] = (W * X_norm.unsqueeze(0)).sum(dim=1)  # [out]

    return importance
```

3. For a given retention ratio r, keep the top-r fraction of dimensions ranked by importance

Output: Per-layer importance tensors that rank every attention head and FFN channel, enabling non-uniform width pruning where sensitive layers retain more capacity.

**Example 3: Evaluate proxy quality on your own model family**

User: "I want to verify that gradient trace correlation actually predicts downstream accuracy for my custom 3B model before committing to a full pruning run."

Approach:
1. Generate 20-30 random pruned architectures at your target compression ratio
2. Score each with the Phi proxy (one forward-backward pass each, ~2 minutes per candidate)
3. For each candidate, run a short continued pretraining (1B tokens) and evaluate on a held-out benchmark
4. Compute Spearman rank correlation between Phi scores and downstream accuracy

```bash
# Pseudocode for proxy validation
for arch in random_architectures:
    phi_score = compute_phi(arch, base_model, calibration_data)
    pruned_model = realize_architecture(arch, base_model)
    train(pruned_model, tokens=1e9)  # Short recovery
    accuracy = evaluate(pruned_model, benchmark="hellaswag,arc,winogrande")
    results.append((phi_score, accuracy))

spearman_rho = scipy.stats.spearmanr(
    [r[0] for r in results],
    [r[1] for r in results]
).correlation
print(f"Proxy-accuracy Spearman rho: {spearman_rho:.3f}")
# Expected: > 0.85 for a well-behaved model family
```

Output: If Spearman rho > 0.85, the proxy is reliable for your model family and full NAS is justified. If below 0.7, investigate whether your model has unusual gradient dynamics (e.g., heavy quantization-aware training artifacts).

## Best Practices

- **Do:** Use activation-weighted magnitude importance (not raw weight magnitude) for width pruning. LLMs have outlier features where activation magnitude varies by 100x across channels — ignoring this produces poor masks.
- **Do:** Standardize gradient traces before computing correlation. The mean-centering and variance normalization is what makes the proxy scale-invariant across heterogeneously pruned blocks.
- **Do:** Weight per-layer correlation by retention ratio when aggregating Phi. Without sparsity weighting, aggressively pruned layers (high variance, low signal) dominate the score.
- **Do:** Round MLP hidden dimensions to multiples of 32 after pruning for hardware-efficient inference on GPUs.
- **Avoid:** Pruning K/V heads in GQA models (Llama-3, Qwen-2.5). This breaks KV-cache sharing and degrades throughput. Only prune Q heads and attention output projections.
- **Avoid:** Using fewer than 16 calibration sequences. The gradient trace estimate becomes noisy and proxy rankings degrade. More than 64 sequences gives diminishing returns.
- **Avoid:** Removing non-contiguous blocks without checking residual stream continuity. Prefer contiguous-range crossover in depth evolution to maintain information flow through skip connections.

## Error Handling

| Problem | Cause | Solution |
|---------|-------|----------|
| Phi scores are all near zero | Calibration data is out-of-distribution | Use data matching the model's pretraining domain |
| OOM during gradient computation | Full gradient storage on large models | Enable LoRA-style low-rank gradient approximation or use gradient checkpointing |
| Proxy ranking disagrees with actual performance | Model has unusual architecture (MoE, hybrid) | Validate proxy with 20-30 candidates before full NAS run (see Example 3) |
| Pruned model outputs gibberish before recovery training | Too aggressive compression (>70% params removed) | Stay within 40-60% retention; ensure at least 60% of layers survive in depth |
| Evolution stagnates (no Phi improvement) | Population too small or mutation rate too low | Increase population to 50, raise bit-flip probability for depth, increase Gaussian jitter variance for width |
| MLP dimensions cause CUDA kernel failures | Non-aligned hidden dimensions | Round all MLP hidden dims to multiples of 32 (or 64 for some architectures) |

## Limitations

- **Requires continued pretraining for recovery.** TraceNAS finds architectures that *will recover well*, but the pruned model still needs ~20B tokens of continued pretraining to reach competitive accuracy. The zero-shot proxy predicts recoverability, not zero-shot performance.
- **Validated primarily on decoder-only transformers.** The method is demonstrated on Llama-2, Llama-3.1, and Qwen-2.5. Applicability to encoder-decoder, MoE, or state-space models is unverified.
- **Single-GPU search assumes model fits in memory.** For models exceeding single-GPU VRAM (e.g., 70B+), you need model parallelism during the search phase, which adds engineering complexity.
- **Proxy correlation may degrade at extreme compression.** At retention ratios below ~30%, gradient traces become sparse and correlation estimates are noisy. The method is best suited for moderate compression (40-70% retention).
- **Does not optimize for latency or throughput directly.** The search maximizes gradient alignment, not inference speed. A high-Phi architecture might have non-uniform widths that reduce hardware utilization compared to uniform pruning.

## Reference

**Paper:** [TraceNAS: Zero-shot LLM Pruning via Gradient Trace Correlation](https://arxiv.org/abs/2602.02891v1) (Malettira et al., 2026)

Key sections to study: Section 3 for the gradient trace correlation proxy formulation and its scale-invariance proof; Section 4 for the evolutionary NAS procedure and search space encoding; Table 1 for proxy validation metrics (Spearman rho = 0.94); Tables 2-3 for benchmark results on Llama and Qwen families.