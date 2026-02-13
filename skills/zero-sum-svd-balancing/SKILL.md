---
name: "zero-sum-svd-balancing"
description: "Compress LLMs using Zero Sum SVD (ZS-SVD) — a post-training low-rank compression method that globally allocates heterogeneous ranks across weight matrices by balancing first-order calibration loss estimates. Use when the user asks to: (1) 'compress an LLM with SVD', (2) 'reduce model size with low-rank factorization', (3) 'apply ZS-SVD to a transformer', (4) 'allocate ranks across layers for SVD compression', (5) 'speed up inference with low-rank decomposition', (6) 'implement activation-whitened SVD compression'."
---

# Zero Sum SVD: Balanced Low-Rank LLM Compression

This skill enables Claude to implement and apply **Zero Sum SVD (ZS-SVD)**, a post-training compression method for large language models. ZS-SVD decomposes each linear layer's weight matrix into low-rank factors using activation-whitened SVD, then performs **global** singular component selection across the entire model. Instead of giving every similarly-sized matrix the same rank, ZS-SVD uses first-order calibration loss estimates to prune components with a zero-sum balancing rule — components whose removal increases loss are paired against components whose removal decreases loss, keeping net predicted loss drift near zero. This yields heterogeneous per-layer ranks automatically without solving an expensive optimization problem.

## When to Use

- When the user wants to compress a pretrained LLM (LLaMA, Mistral, OPT, Vicuna, etc.) for deployment on limited hardware
- When the user asks about SVD-based model compression and needs better rank allocation than uniform/homogeneous approaches
- When the user wants to speed up transformer inference via low-rank weight factorization (up to ~5.8x throughput gains)
- When the user is implementing a compression pipeline and needs to decide how many singular values to keep per layer
- When the user wants a fast alternative to iterative rank-optimization methods like Dobi-SVD (minutes vs. hours)
- When the user asks to reduce memory footprint of attention projections (Q, K, V, O) and MLP layers while preserving accuracy

## Key Technique

**The core problem:** SVD-based LLM compression replaces each weight matrix W (shape m x n) with a rank-k approximation W_k = U_k Sigma_k V_k^T, where k(m+n) < mn saves parameters. The critical question is: given a global compression budget, what rank should each matrix get? Prior methods either assign uniform ranks (ignoring that some layers are far more loss-sensitive than others) or run expensive per-layer optimization (Dobi-SVD takes ~19 hours vs. ZS-SVD's ~16 minutes for LLaMA-7B).

**ZS-SVD's insight:** Work in activation-whitened coordinates. Compute the whitening factor S from calibration-data covariance (C = XX^T, then Cholesky on C + lambda*I). Form A = W*S and take its SVD. In this space, the Eckart-Young theorem gives the optimal low-rank approximation for input reconstruction, not just weight reconstruction. Then estimate each singular component's loss impact via a first-order directional derivative: delta_L_i ~ -sigma_i * (u_i^T H v_i), where H is the whitened gradient from calibration data. Components with positive delta_L (removal helps) and negative delta_L (removal hurts) are placed in separate priority queues.

**The zero-sum rule:** Greedily remove components model-wide, alternating between the two queues to keep cumulative predicted loss change near zero. When the running sum is negative, prefer removing a positive-delta_L component to compensate; when positive, prefer a negative one. Within each queue, pick the smallest-magnitude candidate. This yields heterogeneous ranks — loss-sensitive layers keep more components, insensitive layers get aggressively pruned. An optional post-truncation correction applies a single projected gradient step (which is itself low-rank due to batch size) followed by re-truncation to recover additional accuracy.

## Step-by-Step Workflow

1. **Collect calibration data.** Sample 256 sequences of length 2048 from a representative corpus (WikiText2 is standard). These sequences drive both the whitening covariance estimate and the gradient computation.

2. **Compute per-layer activation covariance.** For each linear layer, run a forward pass over calibration data and accumulate C = X^T X (where X is the layer's input activation matrix). Add ridge regularization: C_reg = C + lambda * I.

3. **Compute whitening factor S.** Perform Cholesky decomposition on C_reg to get S (lower triangular). This transforms the weight space so that reconstruction error in whitened coordinates equals activation-weighted reconstruction error.

4. **Form whitened weights and decompose.** For each weight matrix W, compute A = W @ S, then take the full SVD: A = U Sigma V^T. Store all singular values and vectors.

5. **Compute calibration gradients.** Run a forward-backward pass on calibration data to get the loss gradient with respect to each weight matrix. Transform to whitened coordinates: H = (grad_W L) @ S^{-T}.

6. **Estimate per-component loss sensitivity.** For each singular component i in each layer, compute delta_L_i = -sigma_i * (u_i^T @ H @ v_i). This is a scalar estimating how calibration loss changes if component i is removed.

7. **Run global zero-sum selection.** Pool all components across all layers. Partition into Q_+ (positive delta_L, removal helps) and Q_- (negative delta_L, removal hurts), each as a min-heap sorted by |delta_L|. Enforce per-layer spectral ordering (remove smallest singular values first within each matrix). Greedily remove components, alternating heaps to keep cumulative loss drift near zero, until the global parameter budget is met.

8. **Apply rank thresholding.** For any layer where the resulting rank k satisfies k(m+n) >= m*n, revert to the full dense matrix since the factored form would use more storage.

9. **(Optional) Apply projected gradient correction.** Compute the truncation error delta_W = W_original - W_truncated. Project onto the gradient direction: delta_W' = (<g, delta_W> / <g, g>) * g. Add this correction to W_truncated, then re-truncate to the target rank. Repeat for 1-10 cycles.

10. **Export compressed model.** Store each layer as its (U_k, Sigma_k, V_k^T) factors (or fused U_k @ diag(Sigma_k) and V_k^T for inference). Replace original linear layers with low-rank modules that compute output as x @ V_k^T @ diag(Sigma_k) @ U_k^T.

## Concrete Examples

**Example 1: Compress LLaMA-7B to 60% of original parameters**

User: "I want to compress LLaMA-7B to 60% of its original size using SVD. Can you implement ZS-SVD for this?"

Approach:
1. Load LLaMA-7B and 256 WikiText2 calibration sequences (length 2048)
2. For each of the ~128 linear layers (32 transformer blocks x 4 attention + MLP projections), compute activation covariance and whitening matrix
3. Perform whitened SVD on each weight matrix and compute loss sensitivities
4. Run zero-sum global selection to remove 40% of total singular components, yielding heterogeneous ranks (e.g., attention V projections in early layers may keep rank 3800/4096 while MLP down projections in middle layers drop to rank 1200/4096)
5. Apply 5 cycles of projected gradient correction
6. Export factored weights

Output:
```python
import torch
from transformers import AutoModelForCausalLM, AutoTokenizer

# Step 1: Load model and calibration data
model = AutoModelForCausalLM.from_pretrained("meta-llama/Llama-2-7b-hf", torch_dtype=torch.float32)
tokenizer = AutoTokenizer.from_pretrained("meta-llama/Llama-2-7b-hf")
calib_data = load_wikitext2_calibration(tokenizer, n_samples=256, seq_len=2048)

# Step 2: Collect activation statistics per layer
covariances = {}
hooks = []
for name, module in model.named_modules():
    if isinstance(module, torch.nn.Linear):
        hook = module.register_forward_hook(
            lambda mod, inp, out, n=name: covariances.update(
                {n: covariances.get(n, 0) + inp[0].reshape(-1, inp[0].shape[-1]).T
                 @ inp[0].reshape(-1, inp[0].shape[-1])}
            )
        )
        hooks.append(hook)

# Run calibration forward passes
with torch.no_grad():
    for batch in calib_data:
        model(batch)
for h in hooks:
    h.remove()

# Step 3-4: Whitened SVD + loss sensitivity for each layer
components = []  # (layer_name, index, sigma, delta_L)
for name, module in model.named_modules():
    if isinstance(module, torch.nn.Linear):
        C = covariances[name] / len(calib_data)
        S = torch.linalg.cholesky(C + 1e-4 * torch.eye(C.shape[0]))
        A = module.weight.data @ S  # whitened weight
        U, sigma, Vt = torch.linalg.svd(A, full_matrices=False)
        H = grad_cache[name] @ torch.linalg.inv(S).T  # whitened gradient
        for i in range(len(sigma)):
            delta_L = -sigma[i] * (U[:, i] @ H @ Vt[i, :])
            components.append((name, i, sigma[i].item(), delta_L.item()))

# Step 5: Zero-sum global selection
components_to_remove = zero_sum_select(components, target_ratio=0.6)

# Step 6: Apply truncation and optional correction
apply_truncation(model, components_to_remove, correction_cycles=5)
```

Expected results at 60% compression: WikiText2 perplexity ~20.4 (vs. ~45+ for uniform-rank SVD), mean benchmark accuracy ~0.38.

**Example 2: Compare rank allocations across methods**

User: "Show me how ZS-SVD allocates ranks differently than uniform SVD for a 20% compression ratio."

Approach:
1. Compute whitened SVD for all layers
2. Run zero-sum selection at 80% retention
3. Compare resulting per-layer ranks against uniform allocation
4. Visualize the heterogeneous rank distribution

Output:
```
Layer                    | Uniform Rank | ZS-SVD Rank | delta
-------------------------|-------------|-------------|------
layers.0.attn.q_proj    |    3277     |    3650     | +373  (high sensitivity, preserved)
layers.0.attn.k_proj    |    3277     |    2900     | -377  (low sensitivity, pruned more)
layers.0.mlp.gate_proj  |    3277     |    3500     | +223
layers.15.attn.v_proj   |    3277     |    2100     | -1177 (very low sensitivity)
layers.31.mlp.down_proj |    3277     |    3700     | +423  (final layers preserved)
...
Total params retained: 80% (both methods, same budget — different allocation)
ZS-SVD perplexity: 8.2 | Uniform SVD perplexity: 11.7
```

**Example 3: Implement the zero-sum selection algorithm**

User: "Can you implement just the zero-sum component selection logic?"

Output:
```python
import heapq

def zero_sum_select(components, target_remove_count):
    """
    components: list of (layer_name, sv_index, sigma, delta_L)
    Returns set of (layer_name, sv_index) to remove.
    """
    # Partition by sign of delta_L, use min-heaps on |delta_L|
    Q_pos = []  # removal helps (delta_L > 0)
    Q_neg = []  # removal hurts (delta_L < 0)

    # Group by layer, sort by sigma ascending (remove smallest first)
    from collections import defaultdict
    by_layer = defaultdict(list)
    for name, idx, sigma, dL in components:
        by_layer[name].append((sigma, idx, dL))
    for layer in by_layer:
        by_layer[layer].sort()  # ascending sigma

    # Initialize heaps with the smallest-sigma component per layer
    layer_ptr = {}
    for layer, comps in by_layer.items():
        sigma, idx, dL = comps[0]
        layer_ptr[layer] = 0
        entry = (abs(dL), layer, idx, dL)
        if dL >= 0:
            heapq.heappush(Q_pos, entry)
        else:
            heapq.heappush(Q_neg, entry)

    removed = set()
    cumulative_dL = 0.0

    for _ in range(target_remove_count):
        # Pick heap that pushes cumulative_dL toward zero
        if cumulative_dL <= 0 and Q_pos:
            chosen_heap = Q_pos
        elif cumulative_dL > 0 and Q_neg:
            chosen_heap = Q_neg
        elif Q_pos:
            chosen_heap = Q_pos
        else:
            chosen_heap = Q_neg

        _, layer, idx, dL = heapq.heappop(chosen_heap)
        removed.add((layer, idx))
        cumulative_dL += dL

        # Advance pointer for this layer, push next component
        ptr = layer_ptr[layer] + 1
        comps = by_layer[layer]
        if ptr < len(comps):
            layer_ptr[layer] = ptr
            sigma, next_idx, next_dL = comps[ptr]
            entry = (abs(next_dL), layer, next_idx, next_dL)
            if next_dL >= 0:
                heapq.heappush(Q_pos, entry)
            else:
                heapq.heappush(Q_neg, entry)

    return removed
```

## Best Practices

- **Do:** Always use activation whitening, not raw weight SVD. The whitened decomposition accounts for the actual input distribution and consistently outperforms naive SVD by large margins.
- **Do:** Enforce spectral ordering within each layer — always remove the smallest singular values first before larger ones. The zero-sum rule handles cross-layer decisions, not intra-layer reordering.
- **Do:** Apply the projected gradient correction when targeting aggressive compression (40%+ parameter removal). At mild compression (10-20%), the base method is usually sufficient.
- **Do:** Use at least 256 calibration sequences of length 2048 for stable covariance and gradient estimates. Fewer samples lead to noisy sensitivity scores.
- **Avoid:** Skipping the rank threshold check. If k(m+n) >= m*n for a layer, the factored form is larger than the original dense matrix — store the dense matrix instead.
- **Avoid:** Applying ZS-SVD to embedding or LM head layers. These are typically not good candidates for low-rank compression due to their unique role in the model. Focus on attention projections and MLP layers.

## Error Handling

- **Singular covariance matrix:** If C is near-singular (common for layers with dead neurons), the Cholesky decomposition fails. Increase the ridge regularization lambda (start at 1e-4, increase to 1e-2 if needed).
- **NaN in loss sensitivity:** This occurs when the whitening inverse S^{-T} amplifies numerical noise. Switch to a pseudoinverse with a condition number threshold, or increase lambda.
- **Out-of-memory during full SVD:** For very large layers (e.g., 8192x28672 in 70B models), use randomized SVD (`torch.svd_lowrank`) with an oversampling factor of 10-20. This is approximate but sufficient since low-rank components are being discarded anyway.
- **Correction divergence:** If the projected gradient correction increases perplexity, reduce the number of correction cycles or skip it. This can happen when calibration data is not representative of deployment distribution.
- **Imbalanced heaps:** If nearly all components have the same sign of delta_L, the zero-sum balancing has limited effect. This is rare but can indicate a problem with gradient computation — verify the calibration loss is computed correctly.

## Limitations

- **Post-training only.** ZS-SVD does not fine-tune the model after compression. For maximum accuracy recovery, combine with a few hundred steps of LoRA fine-tuning on the compressed model.
- **Linear layers only.** The method targets nn.Linear weight matrices. LayerNorm parameters, embeddings, and the LM head are not compressed.
- **First-order approximation.** The loss sensitivity estimates are linear approximations. At very aggressive compression (>60% removal), second-order effects dominate and the zero-sum balancing becomes less reliable.
- **Calibration dependence.** The rank allocation is influenced by the calibration dataset. Domain-specific deployment should use domain-representative calibration data, not generic WikiText2.
- **No quantization synergy out of the box.** ZS-SVD produces float factors. Combining with INT8/INT4 quantization of the U and V factors is possible but requires separate implementation.
- **Code availability.** As of this writing, the official repository (github.com/mint-vu/Zero-Sum-SVD) has not yet been populated with code. Implementations must follow the paper's algorithm descriptions.

## Reference

**Paper:** [Zero Sum SVD: Balancing Loss Sensitivity for Low Rank LLM Compression](https://arxiv.org/abs/2602.02848v1) — Abbasi, Thrash, Qin, Sharma, Seifi (2026). Focus on Section 3 (activation whitening + loss sensitivity formulation), Algorithm 1 (zero-sum selection procedure), and Table 1 (compression results across architectures).