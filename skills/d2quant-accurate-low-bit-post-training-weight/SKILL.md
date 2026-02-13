---
name: "d2quant-accurate-low-bit-post-training-weight"
description: "Apply the D2Quant post-training weight quantization framework to compress LLMs to sub-4-bit precision (2-bit, 3-bit) with minimal accuracy loss. Uses Dual-Scale Quantizer (DSQ) for down-projection matrices and Deviation-Aware Correction (DAC) for activation alignment. Trigger phrases: 'quantize LLM to 2-bit', 'apply D2Quant', 'post-training quantization for LLM', 'compress model weights sub-4-bit', 'reduce LLM memory footprint', 'weight-only quantization pipeline'."
---

# D2Quant: Accurate Low-Bit Post-Training Weight Quantization for LLMs

This skill enables Claude to implement and apply the D2Quant weight-only post-training quantization (PTQ) framework, which compresses LLM weights to sub-4-bit precision (2-bit or 3-bit) while preserving accuracy far better than standard approaches like GPTQ or AWQ. D2Quant solves two specific problems: (1) down-projection matrices in FFN blocks are quantization bottlenecks, addressed by a Dual-Scale Quantizer (DSQ) with an absorbable scaling factor; (2) weight quantization causes systematic activation distribution shifts, corrected by Deviation-Aware Correction (DAC) that inserts a mean-shift bias into LayerNorm layers. Together, these yield state-of-the-art sub-4-bit quantization with negligible runtime overhead.

## When to Use

- When the user wants to quantize an LLM (LLaMA, Qwen, Mistral, etc.) to 2-bit or 3-bit precision for deployment on memory-constrained hardware
- When implementing a custom post-training quantization pipeline and needs to handle down-projection matrix sensitivity
- When the user observes accuracy degradation at sub-4-bit quantization and wants a correction strategy
- When building a quantization script that processes transformer blocks sequentially with calibration data
- When the user needs to fold quantization scaling factors into existing weight matrices to avoid inference overhead
- When comparing or integrating D2Quant alongside GPTQ, AWQ, or QuIP# in a quantization toolkit

## Key Technique

**Problem:** Standard weight-only PTQ at sub-4-bit precision suffers from two compounding error sources. First, down-projection matrices (the `W_down` in FFN blocks that project from intermediate dimension back to hidden dimension) have weight distributions that are particularly hard to quantize -- they exhibit high variance across columns, making uniform per-channel quantization lossy. Second, even when individual weight matrices are quantized well in isolation, the cumulative effect on activations flowing through the network causes systematic distribution shifts that compound layer over layer.

**Dual-Scale Quantizer (DSQ):** For down-projection matrices specifically, DSQ introduces a column-wise scaling vector `s^c` that rescales quantized weights: `W_hat = Q(W / s^c) * s^c`. The optimization alternates between (a) solving for `s^c` in closed form given frozen quantized weights (a least-squares problem), and (b) re-quantizing `W / s^c` with the updated scale. This converges within ~15 iterations. The critical insight is that `s^c` can be algebraically absorbed into the preceding up-projection's per-channel scale factors after quantization, so there is zero runtime cost -- the dual scale disappears at inference time. This effectively gives the down-projection a second degree of freedom for fitting without increasing the stored bit budget.

**Deviation-Aware Correction (DAC):** After quantizing each transformer block's attention module, DAC measures the mean activation deviation between full-precision and quantized outputs at the post-attention LayerNorm. It computes a per-dimension bias `mu` from calibration data, then adds this as a correction term inside LayerNorm: `Y_aligned = Y_q + mu`. The correction is most effective on dimensions with high signal-to-noise ratio (SNR = |mu|^2 / sigma^2), where the mean shift is consistent across tokens. The error reduction for each dimension equals `SNR / (1 + SNR)`, so DAC automatically focuses correction where it matters most. The bias is stored as a small additional LayerNorm parameter (one vector per layer, negligible memory).

## Step-by-Step Workflow

1. **Prepare calibration data.** Sample 128 sequences of length 2048 tokens from a representative corpus (WikiText-2 or domain-specific text). Load them through the full-precision model to collect per-block input activations.

2. **Set up block-wise quantization loop.** Process transformer blocks sequentially from layer 0 to layer N-1. For each block, maintain the calibration inputs which get updated after processing each block.

3. **Collect full-precision post-attention LayerNorm activations.** Before quantizing the attention sub-block, run calibration data through the full-precision attention and record the LayerNorm output: `Y_fp = LayerNorm(Attention_fp(X))`.

4. **Quantize attention weight matrices.** Apply per-channel quantization (group size 128) to Q, K, V, and O projection matrices at the target bit-width (2 or 3 bits). Use asymmetric uniform quantization with standard round-to-nearest.

5. **Apply DAC to post-attention LayerNorm.** Run calibration data through the now-quantized attention to get `Y_q`. Compute the per-dimension mean deviation: `mu = mean(Y_fp - Y_q, dim=tokens)`. Store `mu` as a bias correction in the LayerNorm layer. Verify correction by checking that the corrected activation mean is closer to the full-precision mean.

6. **Quantize FFN up-projection and gate-projection.** Apply per-channel quantization to `W_up` and `W_gate` at the target bit-width using standard methods (GPTQ-style or round-to-nearest with group quantization).

7. **Apply DSQ to the down-projection matrix.** Initialize column-wise scale `s^c = ones(1, H)`. Iterate 15 times: (a) quantize `W_down / s^c` to get `Q(W_down / s^c)`, (b) solve for optimal `s^c` via closed-form least-squares: `s^c_j = (Q_j^T W_j) / (Q_j^T Q_j)` per column j. After convergence, compute final quantized down-projection as `W_down_q = Q(W_down / s^c)`.

8. **Fold DSQ scale into up-projection.** Absorb `s^c` into the preceding up-projection's output scaling: multiply the up-projection's per-channel output scales by `s^c`. This eliminates `s^c` from inference. Verify that `W_up_scaled @ activation` followed by `W_down_q` produces equivalent output to the original dual-scale formulation.

9. **Update calibration data.** Forward the calibration inputs through the now-quantized block to produce updated activations for the next block's quantization. This ensures each subsequent block is calibrated against realistic quantized inputs, not idealized full-precision ones.

10. **Evaluate and validate.** After all blocks are quantized, measure perplexity on a held-out validation set (WikiText-2, C4) and run zero-shot benchmarks (MMLU, HellaSwag, ARC, PIQA) to confirm accuracy is within expected bounds.

## Concrete Examples

**Example 1: Quantizing LLaMA-3-8B to 2-bit**

User: "I want to quantize LLaMA-3-8B to 2-bit weights using D2Quant. Set up the pipeline."

Approach:
1. Load model and prepare 128 calibration sequences from WikiText-2
2. Configure quantization: 2-bit asymmetric, group size 128
3. Implement block-wise loop over 32 transformer layers
4. For each layer, apply DAC after attention quantization, DSQ on down-projection

```python
import torch
from transformers import AutoModelForCausalLM, AutoTokenizer

model = AutoModelForCausalLM.from_pretrained("meta-llama/Meta-Llama-3-8B", torch_dtype=torch.float16)
tokenizer = AutoTokenizer.from_pretrained("meta-llama/Meta-Llama-3-8B")

# Calibration data: 128 samples, 2048 tokens each
calib_data = load_calibration_data(tokenizer, dataset="wikitext2", nsamples=128, seqlen=2048)

BIT_WIDTH = 2
GROUP_SIZE = 128
DSQ_ITERATIONS = 15

for block_idx, block in enumerate(model.model.layers):
    # Step 1: Collect full-precision post-attention LayerNorm activations
    with torch.no_grad():
        y_fp = collect_post_attn_layernorm_output(block, calib_data)

    # Step 2: Quantize attention (Q, K, V, O projections)
    for name in ["q_proj", "k_proj", "v_proj", "o_proj"]:
        w = getattr(block.self_attn, name).weight.data
        w_q, scales, zeros = quantize_per_channel(w, bit_width=BIT_WIDTH, group_size=GROUP_SIZE)
        store_quantized(block.self_attn, name, w_q, scales, zeros)

    # Step 3: Apply DAC - compute mean activation deviation
    with torch.no_grad():
        y_q = collect_post_attn_layernorm_output(block, calib_data)
    mu = (y_fp - y_q).mean(dim=0)  # per-dimension mean shift
    block.post_attention_layernorm.bias = torch.nn.Parameter(mu)  # store as LN bias

    # Step 4: Quantize up and gate projections
    for name in ["up_proj", "gate_proj"]:
        w = getattr(block.mlp, name).weight.data
        w_q, scales, zeros = quantize_per_channel(w, bit_width=BIT_WIDTH, group_size=GROUP_SIZE)
        store_quantized(block.mlp, name, w_q, scales, zeros)

    # Step 5: Apply DSQ to down-projection
    w_down = block.mlp.down_proj.weight.data.clone()
    s_col = torch.ones(1, w_down.shape[1], device=w_down.device)

    for iteration in range(DSQ_ITERATIONS):
        w_scaled = w_down / s_col
        w_q_down, _, _ = quantize_per_channel(w_scaled, bit_width=BIT_WIDTH, group_size=GROUP_SIZE)
        # Closed-form solve for s_col per column
        for j in range(w_down.shape[1]):
            q_j = w_q_down[:, j]
            w_j = w_down[:, j]
            s_col[0, j] = (q_j @ w_j) / (q_j @ q_j + 1e-8)

    # Step 6: Fold s_col into up-projection scales
    final_w_q = quantize_per_channel(w_down / s_col, bit_width=BIT_WIDTH, group_size=GROUP_SIZE)
    fold_scale_into_up_proj(block.mlp.up_proj, s_col)
    store_quantized(block.mlp, "down_proj", *final_w_q)

    # Step 7: Update calibration data through quantized block
    calib_data = forward_through_block(block, calib_data)
```

Output: A 2-bit quantized LLaMA-3-8B with ~14 perplexity on WikiText-2 (vs ~20+ for vanilla GPTQ at 2-bit), fitting in ~2.5 GB VRAM for weights alone.

**Example 2: Adding DAC correction to an existing quantization pipeline**

User: "I already have GPTQ quantization working but accuracy drops at 3-bit. Can I add the DAC correction on top?"

Approach:
1. DAC is applied per-block after quantizing the attention sub-layer
2. It only requires collecting pre/post quantization LayerNorm activations
3. The correction is a single bias vector per LayerNorm -- trivial to add

```python
def apply_dac_correction(block, calib_inputs, quantize_attention_fn):
    """Add DAC to any existing quantization pipeline."""
    # Collect full-precision baseline
    y_fp = run_post_attn_layernorm(block, calib_inputs)

    # Run your existing attention quantization
    quantize_attention_fn(block)

    # Collect post-quantization activations
    y_q = run_post_attn_layernorm(block, calib_inputs)

    # Compute and apply mean-shift correction
    deviation = y_fp - y_q  # shape: (n_samples, seq_len, hidden_dim)
    mu = deviation.mean(dim=(0, 1))  # per-dimension mean

    # Compute SNR to verify correction quality
    sigma_sq = deviation.var(dim=(0, 1))
    snr = mu.pow(2) / (sigma_sq + 1e-8)
    expected_reduction = snr / (1 + snr)
    print(f"DAC: mean expected error reduction = {expected_reduction.mean():.3f}")

    # Apply as LayerNorm bias (create if absent)
    ln = block.post_attention_layernorm
    if ln.bias is None:
        ln.bias = torch.nn.Parameter(mu)
    else:
        ln.bias.data += mu
```

Output: Adding DAC alone typically recovers 0.3-1.0 perplexity points at 3-bit, with the largest gains on attention-heavy architectures.

**Example 3: Implementing DSQ scale folding for zero-overhead inference**

User: "How do I make sure the DSQ dual scale doesn't add latency at inference?"

Approach:
1. After DSQ optimization, `s_col` must be absorbed into the up-projection
2. Since `Y = W_down_q * diag(s_col) * X_intermediate`, and `X_intermediate = W_up * X`
3. We fold: `W_up_new = diag(s_col) * W_up`, then `Y = W_down_q * W_up_new * X`

```python
def fold_dsq_scale(up_proj_layer, down_proj_quantized_scales, s_col):
    """Absorb DSQ column scale into up-projection output channels.

    After folding, the down-projection quantized weights can be used
    directly without any additional scaling at inference.
    """
    # s_col shape: (1, intermediate_dim)
    # up_proj output channels correspond to down_proj input channels

    # If up_proj stores per-channel quantization scales, multiply them
    if hasattr(up_proj_layer, 'scales'):
        # Quantized format: dequant(w) = (w_int - zero) * scale
        # We need: dequant(w) * s_col = (w_int - zero) * (scale * s_col)
        up_proj_layer.scales.data *= s_col.squeeze(0)
    else:
        # Full precision: just multiply output dimension
        up_proj_layer.weight.data *= s_col.T  # broadcast across input dim

    # Verify equivalence
    x_test = torch.randn(1, up_proj_layer.weight.shape[1])
    y_original = down_proj_quantized_scales @ (s_col * (up_proj_layer.weight @ x_test.T))
    y_folded = down_proj_quantized_scales @ (up_proj_layer.weight @ x_test.T)
    assert torch.allclose(y_original, y_folded, atol=1e-5), "Folding verification failed"
```

Output: Zero additional parameters or computation at inference. The quantized model has the same weight count and dequantization cost as standard per-channel quantization.

## Best Practices

- **Do:** Always process blocks sequentially and update calibration data after each block. Stale calibration inputs cause error accumulation across layers.
- **Do:** Run DSQ for exactly 15 iterations -- the paper shows convergence plateaus there. Fewer iterations leave accuracy on the table; more iterations waste compute without gains.
- **Do:** Apply DAC specifically to post-attention LayerNorm, not to all LayerNorms. The paper's SNR analysis shows post-attention positions have consistently higher signal-to-noise, making correction most effective there.
- **Do:** Use 128 calibration samples at 2048 token length. This provides sufficient statistical coverage for computing stable mean-shift corrections.
- **Avoid:** Applying DSQ to all weight matrices. It is designed specifically for down-projection matrices where column-wise variance is the bottleneck. Applying it elsewhere adds iteration cost with negligible benefit.
- **Avoid:** Skipping the scale-folding step. If `s_col` is left as a separate runtime multiplication, you add a dense matmul to every forward pass for no reason.
- **Avoid:** Using DAC with very small calibration sets (< 32 samples). The mean-shift estimate becomes noisy and can introduce more error than it corrects.

## Error Handling

- **DSQ divergence:** If `s_col` values explode (> 10x), clamp them to a reasonable range (0.1 to 10.0). This can happen with near-zero quantized column norms -- add epsilon (1e-8) to the denominator in the closed-form solve.
- **LayerNorm without bias:** Some model architectures use bias-free LayerNorm. For DAC, you need to either add a bias parameter or implement the correction as a separate addition module after the LayerNorm.
- **OOM on large models:** For 70B+ models, process one block at a time and offload quantized blocks to CPU. Keep only the current block and calibration data on GPU. Use 3 GPUs for 70B models if available.
- **Accuracy still poor after D2Quant:** At 2-bit, some accuracy loss is unavoidable. If perplexity exceeds 2x the full-precision baseline, verify that (a) calibration data is representative of the deployment domain, (b) group size is 128 (not larger), and (c) both DSQ and DAC are active, not just one.
- **Scale folding mismatch:** After folding `s_col` into up-projection, always run a numerical verification on a test input. Off-by-one dimension indexing between up-projection output channels and down-projection input channels is a common implementation bug.

## Limitations

- **Sub-2-bit is out of scope.** D2Quant targets 2-bit and 3-bit precision. At 1-bit, the quantization grid is too coarse for DSQ's continuous optimization to help meaningfully.
- **Weight-only quantization.** D2Quant does not quantize activations. If your bottleneck is activation memory (long-context inference), you need a separate activation quantization scheme.
- **Calibration data dependency.** The DAC correction biases are computed from calibration data. If deployment-time inputs have very different statistical properties (e.g., code vs. natural language), the correction may be suboptimal.
- **Architecture assumption.** DSQ assumes a standard FFN structure with up-projection followed by down-projection. Non-standard architectures (e.g., MoE experts with shared projections) may need adaptation.
- **No speedup from quantization alone.** Weight-only PTQ reduces memory but does not automatically speed up computation unless paired with a kernel that supports low-bit dequantization (e.g., Marlin, ExLlama). D2Quant produces quantized weights compatible with these kernels but does not include them.

## Reference

- **Paper:** [D2Quant: Accurate Low-bit Post-Training Weight Quantization for LLMs](https://arxiv.org/abs/2602.02546v2) -- Focus on Section 3 (method) for DSQ closed-form derivation and DAC SNR analysis, and Algorithm 1 for the complete block-wise pipeline.
- **Code:** [https://github.com/XIANGLONGYAN/D2Quant](https://github.com/XIANGLONGYAN/D2Quant)