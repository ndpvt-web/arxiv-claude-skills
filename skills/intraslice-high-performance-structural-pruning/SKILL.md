---
name: "intraslice-high-performance-structural-pruning"
description: "Implement IntraSlice block-intra PCA structural pruning for LLMs. Compresses Transformer attention and FFN modules by applying approximate PCA within blocks, fusing transformation matrices into weights with zero extra parameters. Use when: 'prune an LLM with PCA', 'structurally compress a Transformer model', 'reduce LLM size without unstructured sparsity', 'implement IntraSlice pruning', 'compress Llama/Phi model for deployment', 'apply block-intra PCA to shrink a language model'."
---

# IntraSlice: Block-Intra PCA Structural Pruning for LLMs

This skill enables Claude to implement and guide users through IntraSlice, a structural pruning framework that compresses Large Language Models by applying approximate PCA decomposition *within* Transformer modules (attention and FFN blocks) rather than between them. The key innovation is that transformation matrices fuse directly into existing weight matrices with zero additional parameters, avoiding the activation distribution disruption caused by inter-module PCA methods like SliceGPT. This produces genuinely smaller, faster models -- not sparse masks -- with substantially better quality retention at 20-50% compression.

## When to Use

- When the user wants to structurally prune a Llama2, Llama3, or Phi-series model for faster inference
- When the user asks how to compress an LLM using PCA-based methods without adding extra parameters
- When the user needs a global non-uniform pruning strategy that allocates different compression ratios per layer
- When the user wants to reduce attention head count or FFN intermediate dimension in a Transformer
- When the user is comparing pruning approaches (SliceGPT, SoBP, LaCo) and wants the IntraSlice method
- When the user asks to prepare a pruned model for deployment on constrained hardware (single GPU, edge)
- When the user wants to understand or implement approximate PCA that handles nonlinearities in MLP gates

## Key Technique

**The Problem with Inter-Module PCA:** Prior methods like SliceGPT apply PCA transformations between Transformer modules. Because outputs pass through residual connections, the PCA transformation must be computed online at every residual add, introducing extra parameters at each connection and causing cumulative activation distribution drift across layers. At 30% sparsity on LLaMA2-7B, SliceGPT degrades to 8.64 perplexity vs. 5.47 baseline.

**IntraSlice's Intra-Module Approach:** Instead of transforming activations between modules, IntraSlice applies approximate PCA compression strictly *within* each MHA and FFN block. For MHA, this means compressing the Value/Output pathway via a least-squares regression matrix `Q2* = ((XQ2)^T(XQ2) + lambda*I)^(-1) (XQ2)^T X`, and using pairwise channel selection for Query/Key (compatible with RoPE). For FFN, the Up/Gate/Down projections are jointly compressed so the transformation absorbs directly into the weight matrices. Because compression happens inside the module, the output perturbation is small relative to the residual stream, preserving activation distributions across layers.

**Global Pruning Ratio Estimation:** Rather than applying uniform compression, IntraSlice computes per-layer importance scores using PCA-aware gradient corrections: `I_i^l = ((g^l * Q_s^l)_i)^2`, where `Q_s^l` is a sparse PCA transformation capturing how gradients interact with the compressed subspace. A bias parameter `lambda_b` scales MLP pruning ratios relative to MHA, and the allocator distributes a target sparsity budget non-uniformly across all layers. This consistently outperforms uniform allocation, especially at high compression ratios (40-50%).

## Step-by-Step Workflow

1. **Select the target model and target sparsity.** Load the pretrained model (Llama2-7B/13B/70B, Llama3-8B, Phi-3, or similar decoder-only Transformer) via HuggingFace Transformers. Choose a target sparsity between 20-50% (30% is a safe starting point; beyond 50% quality degrades significantly).

2. **Prepare calibration data.** Sample 128 sequences of length 2048 from WikiText2 training set (or C4). Run a forward pass collecting activations at each module's input and output boundaries. Store these as calibration tensors -- they drive the PCA computation and importance scoring.

3. **Compute per-layer importance scores with PCA-aware gradients.** For each layer, compute the gradient of reconstruction loss with respect to module outputs. Apply the sparse PCA transformation to these gradients: `I_i^l = ((g^l * Q_s^l)_i)^2`. This captures not just raw importance but how well each dimension survives compression.

4. **Allocate non-uniform pruning ratios globally.** Using the importance scores, solve the allocation problem: distribute the target sparsity budget across all layers and module types (MHA vs FFN). Apply the bias parameter `lambda_b` to scale MLP ratios relative to MHA ratios. Layers with higher importance scores receive lower pruning ratios.

5. **Compress MHA modules per layer.** For each attention block:
   - **Value/Output path:** Compute the approximate PCA matrix `Q2*` via regularized least-squares regression on calibration activations. This removes the orthogonality constraint to handle the nonlinear interaction. Fuse `Q2*` into `W_v` and `W_o` weight matrices directly.
   - **Query/Key path:** Apply pairwise channel selection (pairs of channels are kept or removed together to preserve RoPE positional encoding structure). For GQA models, synchronize Q/K compression within each group.
   - **Head pruning:** Optionally remove entire attention heads whose reconstruction contribution is lowest, using greedy removal scored by per-head reconstruction error.

6. **Compress FFN modules per layer.** For each feed-forward block:
   - Compute the joint PCA compression across Up-projection (`W_u`), Gate-projection (`W_g`), and Down-projection (`W_d`).
   - Fuse the transformation matrix into `W_u` and `W_g` (input side) and `W_d` (output side), reducing the intermediate dimension.
   - If the layer has a high pruning ratio (>35%), apply sliced iterative refinement: alternate between optimizing the compression and reconstruction matrices in slices to avoid memory blowup.

7. **Fuse all transformation matrices into weights.** Verify that no standalone transformation matrices remain in the computation graph. The compressed model should have the same architectural structure as the original but with smaller weight dimensions -- no extra `nn.Linear` layers or online PCA ops at inference time.

8. **Validate the pruned model.** Evaluate perplexity on WikiText2 test set and run zero-shot accuracy on standard benchmarks (ARC-e, ARC-c, PIQA, WinoGrande, HellaSwag, MMLU). Compare against the unpruned baseline and against SliceGPT/SoBP at the same sparsity. Expected: at 30% sparsity on LLaMA2-7B, perplexity ~7.11 (vs. 8.64 for SliceGPT).

9. **Profile inference speed.** Measure wall-clock latency and throughput on target hardware. Structural pruning yields real speedups (unlike unstructured sparsity which requires sparse kernels). Smaller weight matrices directly reduce matmul FLOPs.

10. **Export the pruned model.** Save the compressed model in HuggingFace format with updated config (reduced `hidden_size`, `intermediate_size`, `num_attention_heads` as appropriate). The model is now a standard Transformer -- no custom ops needed for inference.

## Concrete Examples

**Example 1: Pruning LLaMA2-7B to 30% sparsity**

User: "I want to compress LLaMA2-7B by 30% using IntraSlice for faster inference on a single A800 GPU."

Approach:
1. Load `meta-llama/Llama-2-7b-hf` and prepare 128 calibration samples from WikiText2 (seq_len=2048).
2. Run forward pass to collect activations at every MHA and FFN module boundary.
3. Compute PCA-aware importance scores per layer. Allocate pruning ratios non-uniformly (shallow layers get less pruning, deep layers more, with `lambda_b` biasing MLP compression).
4. For each of the 32 layers:
   - MHA: Compute approximate PCA for V/O path. Fuse into `W_v` (4096x4096 -> 4096x2867) and `W_o` (4096x4096 -> 2867x4096). Apply pairwise channel selection on Q/K.
   - FFN: Jointly compress Up/Gate/Down, reducing intermediate dim from 11008 to ~7706.
5. Validate: expect WikiText2 PPL ~7.11, avg zero-shot accuracy ~60.5%.

Output:
```
Model: LLaMA2-7B -> LLaMA2-7B-pruned-30pct
Original params: 6.74B -> Pruned params: ~4.72B
WikiText2 PPL: 5.47 -> 7.11
Avg zero-shot accuracy: 63.68 -> 60.49
Compression time: ~0.52 hours on A800
Inference speedup: ~1.3x (real wall-clock, batch=1)
Extra parameters added: 0
```

**Example 2: Implementing the approximate PCA fusion for an FFN block**

User: "Show me how to implement the approximate PCA compression for a single FFN module and fuse it into the weights."

Approach:
1. Collect calibration activations `X` (shape: [N, hidden_dim]) at the FFN input.
2. Compute the covariance and solve for the compression matrix.
3. Fuse into Up/Gate/Down projections.

Output:
```python
import torch
import torch.nn as nn

def compress_ffn_intraslice(ffn_module, calibration_input, target_dim, lambda_reg=1e-4):
    """
    Compress an FFN block using IntraSlice approximate PCA.

    Args:
        ffn_module: nn.Module with .up_proj, .gate_proj, .down_proj
        calibration_input: Tensor [num_samples * seq_len, hidden_dim]
        target_dim: reduced intermediate dimension
        lambda_reg: regularization for least-squares
    """
    X = calibration_input.float()  # [N, d_model]

    # Compute activations through up and gate projections
    up_out = X @ ffn_module.up_proj.weight.T.float()    # [N, d_inter]
    gate_out = X @ ffn_module.gate_proj.weight.T.float() # [N, d_inter]
    H = torch.nn.functional.silu(gate_out) * up_out      # [N, d_inter]

    # PCA on intermediate activations to find top components
    # Use covariance method for efficiency
    cov = (H.T @ H) / H.shape[0]  # [d_inter, d_inter]
    eigenvalues, eigenvectors = torch.linalg.eigh(cov)
    # Take top-k eigenvectors (largest eigenvalues are last)
    Q_compress = eigenvectors[:, -target_dim:]  # [d_inter, target_dim]

    # Approximate PCA: relax orthogonality via least-squares reconstruction
    H_compressed = H @ Q_compress  # [N, target_dim]
    # Solve for reconstruction: Q_reconstruct = (H_c^T H_c + lambda I)^-1 H_c^T H
    A = H_compressed.T @ H_compressed + lambda_reg * torch.eye(target_dim, device=X.device)
    B = H_compressed.T @ H
    Q_reconstruct = torch.linalg.solve(A, B)  # [target_dim, d_inter]

    # Fuse into weights: new_down = W_down @ Q_reconstruct^T
    # new_up = Q_compress^T @ W_up, new_gate = Q_compress^T @ W_gate
    with torch.no_grad():
        new_up_weight = (Q_compress.T @ ffn_module.up_proj.weight.float()).to(ffn_module.up_proj.weight.dtype)
        new_gate_weight = (Q_compress.T @ ffn_module.gate_proj.weight.float()).to(ffn_module.gate_proj.weight.dtype)
        new_down_weight = (ffn_module.down_proj.weight.float() @ Q_reconstruct.T).to(ffn_module.down_proj.weight.dtype)

    # Replace modules with smaller linear layers
    d_model = ffn_module.up_proj.weight.shape[1]
    ffn_module.up_proj = nn.Linear(d_model, target_dim, bias=False)
    ffn_module.gate_proj = nn.Linear(d_model, target_dim, bias=False)
    ffn_module.down_proj = nn.Linear(target_dim, d_model, bias=False)

    ffn_module.up_proj.weight.data = new_up_weight
    ffn_module.gate_proj.weight.data = new_gate_weight
    ffn_module.down_proj.weight.data = new_down_weight

    return ffn_module
```

**Example 3: Comparing IntraSlice vs SliceGPT for deployment decision**

User: "Should I use IntraSlice or SliceGPT to compress Llama3-8B for production serving?"

Approach:
1. Compare at equivalent sparsity levels using published benchmarks.
2. Evaluate parameter overhead differences.
3. Consider deployment complexity.

Output:
```
Comparison at 30% sparsity on LLaMA-family models:

| Metric                    | SliceGPT       | IntraSlice     |
|---------------------------|----------------|----------------|
| WikiText2 PPL (LLaMA2-7B)| 8.64           | 7.11           |
| Avg zero-shot accuracy    | 46.70          | 60.49          |
| Extra parameters per layer| Yes (Q matrices in residual) | None  |
| Activation distortion     | Cumulative across layers     | Minimal (intra-module only) |
| RoPE compatibility        | Requires workaround          | Native pairwise selection   |
| Inference overhead        | Online PCA transforms        | Zero (fully fused)          |
| Deployment format         | Custom forward pass needed   | Standard HuggingFace model  |

Recommendation: IntraSlice. The quality gap is large (~14 accuracy points),
there are no extra parameters, and the pruned model is a standard Transformer
that works with any serving framework (vLLM, TGI, TensorRT-LLM) without
custom kernels or modified forward passes.
```

## Best Practices

- **Do:** Start with 20-30% sparsity and validate quality before going higher. The quality-compression tradeoff is smooth up to ~40% but degrades rapidly beyond 50%.
- **Do:** Use the global non-uniform pruning allocator rather than applying the same ratio to every layer. Shallow layers are more sensitive to pruning -- the allocator will automatically protect them.
- **Do:** Apply the iterative sliced refinement for layers assigned high pruning ratios (>35%). This adds compression time but significantly improves reconstruction quality for heavily pruned layers.
- **Do:** Keep calibration data representative of your deployment domain. WikiText2 works for general-purpose models, but domain-specific calibration improves results for specialized deployments.
- **Avoid:** Applying inter-module PCA (SliceGPT-style) and intra-module PCA simultaneously. The residual disruption from inter-module transforms undermines the stability that intra-module compression provides.
- **Avoid:** Pruning Query/Key channels independently when using RoPE. Always use pairwise channel selection (remove channels in pairs) to preserve the rotary position encoding structure. Breaking RoPE pairs causes disproportionate quality loss.
- **Avoid:** Skipping the regularization term `lambda*I` in the least-squares solve. Without it, the regression becomes numerically unstable, especially for layers with near-singular activation covariances.

## Error Handling

- **Out-of-memory during calibration:** Reduce calibration batch size or accumulate covariance matrices incrementally. The PCA computation needs `O(d^2)` memory for covariance, not `O(N*d)` for full activation storage.
- **Numerical instability in least-squares solve:** Increase `lambda_reg` (try 1e-3 or 1e-2). If eigenvalues of the covariance span many orders of magnitude, consider using `torch.linalg.lstsq` instead of explicit inverse.
- **Quality collapse at one specific layer:** The global allocator may have assigned too aggressive a ratio. Manually cap that layer's pruning ratio at 25% and redistribute the budget to less sensitive layers.
- **RoPE degradation after Q/K pruning:** Verify that channels are being removed in pairs (indices 2i and 2i+1 together). Single-channel removal breaks the sin/cos pairing.
- **GQA model dimension mismatch:** For grouped-query attention models (Llama3, Phi-3), ensure Q/K compression is synchronized within each KV group. The number of removed channels must be consistent across all queries sharing a KV head.

## Limitations

- **Decoder-only Transformers only.** The method is validated on causal LMs (Llama, Phi). Encoder-decoder architectures (T5, BART) have different residual connection patterns and cross-attention modules that would require adaptation.
- **No fine-tuning step.** IntraSlice is a post-training compression method. For maximum quality recovery, combining it with a short fine-tuning phase (LoRA or full) could help but is not part of the published method.
- **Single calibration dataset.** Quality depends on calibration data representativeness. Domain shift between calibration and deployment data will reduce the effectiveness of the PCA basis selection.
- **Diminishing returns beyond 50% sparsity.** At extreme compression ratios, the method still outperforms baselines but absolute quality becomes poor (PPL > 15 on LLaMA2-7B). For >50% compression, consider combining with quantization rather than pruning alone.
- **No official reference implementation.** As of the paper's publication, no public code repository is available. Implementations must be built from the paper's algorithmic descriptions.

## Reference

**Paper:** [IntraSlice: Towards High-Performance Structural Pruning with Block-Intra PCA for LLMs](https://arxiv.org/abs/2602.01975v1) (Li et al., 2026). Focus on Section 3 (method) for the approximate PCA formulation and fusion procedure, and Section 4.2 for the global pruning ratio estimator derivation.