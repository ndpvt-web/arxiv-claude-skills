---
name: "pop-prefill-only-pruning-inference"
description: |
  Implement POP (Prefill-Only Pruning) to accelerate LLM and VLM inference by skipping deep transformer layers during the prefill stage while keeping the full model for decode. Achieves up to 1.37x prefill speedup with minimal accuracy loss.
  Trigger phrases:
  - "Speed up LLM prefill latency"
  - "Optimize transformer inference with layer pruning"
  - "Implement stage-aware pruning for LLM serving"
  - "Reduce time-to-first-token for long context inputs"
  - "Apply POP pruning to Llama/Qwen/Gemma"
  - "Skip layers during prefill without losing accuracy"
---

# POP: Prefill-Only Pruning for Efficient LLM Inference

This skill enables Claude to implement and apply Prefill-Only Pruning (POP), a stage-aware inference optimization that exploits a key asymmetry in transformer models: deep layers are critical for next-token prediction (decode) but largely redundant for context encoding (prefill). By skipping the last ~1/3 of layers during the compute-heavy prefill stage, then restoring the full model for decode, POP delivers up to 1.37x prefill speedup with under 3% accuracy loss on most benchmarks -- without any retraining.

## When to Use

- When the user wants to reduce time-to-first-token (TTFT) for long-context LLM serving (512+ tokens)
- When deploying Llama-3.1, Qwen3-VL, Gemma-3, or similar decoder-only transformers and prefill is the bottleneck
- When the user asks to implement structured pruning that doesn't collapse on generation tasks (unlike SliceGPT/ShortGPT)
- When optimizing a multimodal VLM pipeline where image tokens create long prefill sequences
- When the user needs a training-free inference speedup that preserves model weights
- When building a custom inference engine and wants to add stage-aware layer skipping
- When evaluating the accuracy-latency tradeoff of pruning ratios (20%-50%) for a specific deployment

## Key Technique

**The Core Insight: Stage Asymmetry.** Traditional structured pruning removes layers permanently, which devastates autoregressive generation because deep layers carry disproportionate importance for next-token prediction. POP's virtual gate analysis (scalar gates on residual branches, scored via gradient-squared Fisher Information on 200 calibration samples) reveals that these same deep layers contribute minimally during context encoding. This means you can safely skip them during prefill only.

**Independent KV Projections.** The challenge with skipping layers during prefill is that the decode stage expects a complete KV cache for all layers. POP solves this by computing KV projections independently in skipped layers -- applying just the K and V projection matrices (W_K, W_V) to the input hidden state, storing the result, but skipping the full attention and FFN computation. This adds <5% overhead compared to the full layer and gives decode everything it needs.

**Boundary Handling.** The final input token is processed using the full model (as if it were the first decode step), not the pruned prefill path. This prevents error accumulation at the critical transition point and ensures the first generated token has full model capacity behind it. In practice: prefill tokens 1..N-1 use the pruned model, token N uses the full model, then decode continues with the full model.

## Step-by-Step Workflow

1. **Identify the target model architecture.** Determine the total number of transformer layers (L). For Llama-3.1-8B this is 32, for Qwen3-VL-8B it is 24, for Gemma-3-12B it is 12. Locate the layer loop in the model's forward pass (e.g., `for layer in self.layers`).

2. **Choose a pruning ratio.** Default to 1/3 of layers (the paper's sweet spot). For a 32-layer model, skip the last 10-11 layers during prefill. If latency matters more than accuracy, try 25% (safer) or 50% (aggressive). Consult the tradeoff table:
   - 20% pruning: ~1.19x speedup, negligible accuracy loss
   - 25% pruning: ~1.25x speedup, <1% loss on most tasks
   - 33% pruning: ~1.37x speedup, 1-3% loss (recommended default)
   - 50% pruning: ~1.67x speedup, significant degradation on reasoning tasks

3. **Add a stage flag to the forward pass.** Introduce a boolean `is_prefill` parameter (or detect it from the input shape: `is_prefill = input_ids.shape[1] > 1`). This flag controls whether pruned layers execute fully or are skipped.

4. **Implement independent KV projections for pruned layers.** In each layer designated for pruning, when `is_prefill=True`:
   - Compute `K = x @ W_K` and `V = x @ W_V` using the layer's existing projection weights
   - Store these in the KV cache at the correct layer index
   - Skip the attention computation, FFN, and residual addition entirely
   - Pass the input hidden state through unchanged (identity mapping)

5. **Implement boundary handling for the last prefill token.** Split the prefill into two passes:
   - Tokens 1..N-1: run through the pruned model (deep layers skipped)
   - Token N (the last input token): run through the full model with all layers active
   This ensures the first generated token benefits from full model capacity.

6. **Wire up the decode stage normally.** During autoregressive generation (`is_prefill=False`), all layers execute as usual. The KV cache from step 4 is already populated, so attention over prefill positions works correctly.

7. **(Optional) Run importance analysis to select which layers to prune.** Instead of defaulting to the last 1/3, compute per-layer importance scores:
   - Add a scalar gate `g_l` (initialized to 1.0) multiplying each layer's residual output
   - Run forward-backward on 200 calibration samples, sampling targets from the model's own distribution
   - Score each layer as `I_l = E[(dL/dg_l)^2]`
   - Prune layers with the lowest prefill-stage importance scores

8. **Benchmark prefill latency and accuracy.** Measure TTFT at representative sequence lengths (512, 1024, 2048). Verify accuracy on task-specific benchmarks (GSM8K for math, MMLU for knowledge, HumanEval for code). Expect meaningful speedups only at 512+ tokens where prefill is compute-bound.

9. **Integrate into the serving framework.** For vLLM, TGI, or custom engines: hook the stage-aware logic into the existing prefill/decode split. Most serving frameworks already separate these stages, making POP a natural addition.

## Concrete Examples

**Example 1: Adding POP to a HuggingFace Llama model**

User: "I'm serving Llama-3.1-8B and prefill is my bottleneck for 2048-token contexts. Help me implement POP to speed it up."

Approach:
1. Identify that Llama-3.1-8B has 32 layers. At 33% pruning, skip layers 22-32 during prefill.
2. Modify the model's forward method to accept a stage flag.
3. Implement the layer-skipping logic with independent KV projections.

Output (key code modification in the model forward pass):

```python
# In the model's forward loop (e.g., LlamaModel.forward)
POP_SKIP_LAYERS = set(range(22, 32))  # Last ~1/3 of 32 layers
is_prefill = input_ids.shape[1] > 1

for idx, layer in enumerate(self.layers):
    if is_prefill and idx in POP_SKIP_LAYERS:
        # Independent KV projection only -- skip attention + FFN
        k_proj = layer.self_attn.k_proj(hidden_states)
        v_proj = layer.self_attn.v_proj(hidden_states)
        # Store in KV cache at this layer's position
        past_key_values[idx] = (k_proj, v_proj)
        # Identity mapping: hidden_states passes through unchanged
    else:
        hidden_states, kv = layer(
            hidden_states,
            past_key_values=past_key_values[idx],
        )
        past_key_values[idx] = kv
```

For boundary handling, split the prefill:

```python
if is_prefill and input_ids.shape[1] > 1:
    # Process tokens 1..N-1 with pruning
    prefill_hidden = forward_with_pop(input_ids[:, :-1], pop_enabled=True)
    # Process token N with full model
    final_hidden = forward_with_pop(input_ids[:, -1:], pop_enabled=False,
                                     past_key_values=past_key_values)
```

**Example 2: Determining optimal pruning ratio for a deployment**

User: "I want to use POP on Gemma-3-12B but I need to keep accuracy above 95% of baseline on MMLU. What pruning ratio should I use?"

Approach:
1. Gemma-3-12B has 12 layers. Test ratios: 2 layers (17%), 3 layers (25%), 4 layers (33%).
2. Run importance analysis on 200 calibration samples to identify the least important layers.
3. Benchmark each ratio against MMLU.

Output:
```
Importance analysis results (layer scores, lower = safer to prune):
Layer 9:  0.0023  <- prune first
Layer 10: 0.0031
Layer 11: 0.0045
Layer 12: 0.0089
Layer 8:  0.0124
...
Layer 1:  0.0891  <- most important, never prune

Recommendation: Skip layers 10-12 (25% pruning)
- Expected MMLU: ~97% of baseline (within your 95% threshold)
- Expected prefill speedup: ~1.25x at 2048 tokens
- Skipping layer 9 as well (33%) risks dropping below 95% on reasoning tasks
```

**Example 3: Applying POP to a multimodal VLM pipeline**

User: "My Qwen3-VL-8B deployment processes high-res images (2560x1440) and prefill takes too long. Can POP help?"

Approach:
1. High-res images in Qwen3-VL produce long token sequences (image patches), making prefill the dominant cost.
2. Qwen3-VL has 24 layers. At 33% pruning, skip layers 17-24 during prefill.
3. The image encoder runs separately and is unaffected -- POP only modifies the transformer decoder layers.

Output:
```
Configuration for Qwen3-VL-8B with POP:
- Layers to skip during prefill: 17-24 (8 layers, 33%)
- Image encoder: unchanged (runs before prefill)
- Expected speedup: 1.16-1.19x on image inputs
- Accuracy impact: <1% on ScreenSpot, ~1.3% on MMMU

Note: Multimodal speedups are lower than text-only because
the image encoder and vision-language projection add fixed
overhead that POP does not reduce. The 1.19x applies only
to the transformer decoder prefill portion.
```

## Best Practices

- **Do:** Default to 33% pruning ratio (last 1/3 of layers) as the starting point. This is the paper's recommended sweet spot across all tested architectures.
- **Do:** Always implement boundary handling for the last prefill token. Skipping it causes measurable first-token accuracy degradation that compounds through the rest of generation.
- **Do:** Verify that KV cache shapes match between the pruned prefill path and the full decode path. Mismatches cause silent correctness bugs.
- **Do:** Benchmark at realistic sequence lengths. POP shows minimal benefit below 256 tokens because short prefills are memory-bound, not compute-bound.
- **Avoid:** Pruning more than 50% of layers. The paper shows catastrophic failure (GSM8K drops from 80% to 38%) beyond this threshold.
- **Avoid:** Applying POP to the shallow layers (first 1/3). These encode fundamental representations and are important for both prefill and decode. Always prune from the deep end.

## Error Handling

- **KV cache dimension mismatch:** If the pruned-layer KV projections produce tensors with different shapes than the full attention layer would, ensure you're using the same W_K and W_V weight matrices. The projection must match what decode expects.
- **First token quality collapse:** If the first generated token is noticeably worse, boundary handling is missing or broken. Verify the last input token runs through all layers.
- **No speedup observed:** At short sequence lengths (<256 tokens) or small batch sizes, prefill may be memory-bound rather than compute-bound. POP only helps compute-bound prefills. Check with profiling (e.g., `torch.profiler`).
- **Accuracy drops larger than expected:** The calibration data may not represent the deployment distribution. Re-run importance analysis with domain-representative samples. Also verify you're not accidentally skipping layers during decode.
- **OOM during boundary handling:** The boundary pass for the last token uses the full model. If memory is tight, ensure the prefill KV cache for pruned layers is computed in-place without retaining intermediate activations.

## Limitations

- **Short sequences see little benefit.** Below ~256 tokens, prefill is memory-bound and layer skipping provides marginal speedup (1.02-1.05x).
- **Decode speed is unchanged.** POP only accelerates prefill. If decode is your bottleneck (e.g., long generation with short prompts), POP will not help. Consider speculative decoding or KV cache compression instead.
- **Reasoning-heavy tasks are more sensitive.** GSM8K and HumanEval show 2-5% drops at 33% pruning, while factual recall (MMLU) drops only 1-2%. Adjust the pruning ratio downward for math/code workloads.
- **Not validated on models beyond ~12B parameters.** The paper tests 8B and 12B models. Scaling behavior to 70B+ is unknown -- deeper models may tolerate higher pruning ratios, or the importance distribution may shift.
- **Requires inference framework modifications.** This is not a drop-in config change; you need to modify the forward pass, KV cache management, and stage detection logic.

## Reference

**Paper:** [POP: Prefill-Only Pruning for Efficient Large Model Inference](https://arxiv.org/abs/2602.03295v1) (He et al., 2026)
**Key finding to look for:** Table 4 (pruning ratio ablation) and Figure 3 (per-layer importance scores showing the prefill/decode asymmetry) are the most actionable parts for implementation decisions.