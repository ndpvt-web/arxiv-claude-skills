---
name: "towards-understanding-best-practices"
description: "Quantize vision-language models (VLMs) component-by-component using optimal bit-width strategies derived from multimodal quantization research. Guides selection of quantization method (GPTQ, AWQ, RTN), bit width (2-8 bit), and per-component configuration for BLIP-2, LLaVA, and similar multimodal pipelines. Triggers: 'quantize a vision-language model', 'reduce VLM memory usage', 'optimize multimodal model for deployment', 'GPTQ vs AWQ for VLM', 'quantize LLaVA or BLIP-2', 'efficient VLM inference setup'"
---

# Component-Aware Quantization for Vision-Language Models

This skill enables Claude to guide practitioners through quantizing multimodal vision-language models (VLMs) by treating each pipeline component -- vision encoder (ViT), language model (LLM), and connector/projection layers -- as independent quantization targets with distinct sensitivity profiles. Rather than applying uniform bit-width reduction, this approach uses findings from systematic experimentation across GPTQ, AWQ, and RTN methods at 2-bit through 8-bit precision to recommend Pareto-optimal configurations that minimize memory and latency while preserving accuracy on captioning, retrieval, and VQA tasks.

## When to Use

- When the user wants to deploy a vision-language model (BLIP-2, LLaVA, InstructBLIP, or similar) on memory-constrained hardware (consumer GPUs, edge devices)
- When the user asks which quantization method to use for a multimodal model and needs a comparison of GPTQ vs AWQ vs RTN
- When the user needs to reduce VRAM usage of a VLM while keeping acceptable accuracy on captioning, VQA, or retrieval
- When the user is building an inference pipeline and wants to know which components (ViT, LLM, connector) to quantize first and to what precision
- When the user is evaluating trade-offs between model size reduction and task performance for a multimodal system
- When the user asks about mixed-precision or non-uniform quantization strategies for multi-component models

## Key Technique

Traditional quantization applies a single bit-width uniformly across all model parameters. For multimodal pipelines, this is suboptimal because the three components -- vision encoder, language model, and connector -- have fundamentally different parameter counts, architectures, and sensitivity to precision loss. The core insight from this research is that **ViT and LLM contribute comparably to overall model performance despite the LLM containing far more parameters**. This means the vision encoder deserves equal quantization attention even though it is a fraction of the total model size.

The study systematically evaluates three post-training quantization (PTQ) methods across all pipeline components at multiple bit widths (2, 3, 4, 8-bit) against 16-bit baselines. **GPTQ** uses second-order weight update information (approximate Hessian) to minimize quantization error layer by layer. **AWQ** identifies salient weight channels by analyzing activation magnitudes and scales weights before quantization to protect important channels. **RTN** (Round-To-Nearest) is the simplest baseline that rounds each weight to the nearest quantized value. Results show that AWQ generally provides the best accuracy-efficiency trade-off, GPTQ is competitive at 4-bit and above, and RTN degrades rapidly below 4-bit.

The practical recommendation is a **tiered quantization strategy**: the LLM component tolerates aggressive quantization (4-bit AWQ or GPTQ) well, yielding the largest memory savings due to its size; the ViT is more sensitive per-parameter but smaller, so 8-bit quantization preserves its critical feature extraction capability; the connector/projection layers should remain at 8-bit or higher since they are small and act as the information bottleneck between modalities. This mixed-precision approach achieves near-full-precision accuracy at substantially reduced bits-per-weight (bpw).

## Step-by-Step Workflow

1. **Identify the VLM architecture and its components.** Determine which model family is being quantized (BLIP-2, LLaVA, InstructBLIP, etc.) and map its three pipeline stages: vision encoder (e.g., ViT-G, CLIP ViT-L), connector (e.g., Q-Former, linear projection, MLP), and language model (e.g., OPT, Vicuna, LLaMA).

2. **Profile the baseline model.** Measure full-precision (FP16/BF16) VRAM usage, inference latency, and task accuracy on the target benchmarks (captioning with CIDEr/BLEU, VQA accuracy, retrieval with Recall@K). This establishes the ceiling to measure degradation against.

3. **Select quantization method based on constraints.** Choose AWQ for the best accuracy-efficiency trade-off when calibration data is available. Choose GPTQ when you need broad library support (e.g., `auto-gptq`, `transformers` integration). Use RTN only as a quick baseline or when no calibration data is available.

4. **Prepare calibration data.** For GPTQ and AWQ, assemble 128-512 representative samples from the target domain. For multimodal models, calibration should include both image-text pairs (not text-only), so the quantization process accounts for vision-conditioned activations.

5. **Quantize the LLM component first at 4-bit.** The LLM holds the majority of parameters (often 80%+), so quantizing it yields the largest memory reduction. Apply 4-bit AWQ or GPTQ with group size 128. Evaluate accuracy -- if degradation exceeds your tolerance (typically >2% on the primary metric), try 4-bit with smaller group size (64) or fall back to 8-bit.

6. **Quantize the ViT component at 8-bit.** Apply 8-bit quantization (INT8 via RTN or dynamic quantization). The vision encoder is more sensitive per parameter -- dropping below 8-bit often causes disproportionate accuracy loss on retrieval and fine-grained VQA tasks. Only attempt 4-bit ViT quantization if memory is extremely constrained and you accept >3-5% accuracy loss.

7. **Keep the connector at 8-bit or FP16.** The connector is typically small (< 5% of total parameters) and serves as the information bridge. Quantizing it aggressively provides negligible memory savings but risks significant accuracy loss. Leave it at 8-bit minimum.

8. **Evaluate the mixed-precision configuration end-to-end.** Run the full pipeline on all target benchmarks. Compare against the FP16 baseline and a uniform 4-bit baseline. The mixed configuration (4-bit LLM + 8-bit ViT + 8-bit connector) should outperform uniform 4-bit while using nearly the same memory.

9. **Compute the effective bits-per-weight (bpw) and memory footprint.** Calculate: `effective_bpw = sum(component_params * component_bits) / total_params`. Report VRAM reduction as a percentage of baseline. A typical 4/8/8 configuration achieves ~4.5-5.0 effective bpw with <2% accuracy loss.

10. **Package the quantized model for deployment.** Export using the appropriate format: GPTQ models via `auto-gptq` or `transformers` with `GPTQConfig`, AWQ models via `autoawq` or `transformers` with `AwqConfig`. Verify the exported model produces identical outputs to the in-memory quantized version.

## Concrete Examples

**Example 1: Quantizing LLaVA-1.5-13B for consumer GPU deployment**

User: "I want to run LLaVA-1.5-13B on a 24GB GPU for image Q&A. It doesn't fit in FP16. How should I quantize it?"

Approach:
1. Map components: CLIP ViT-L/14-336px (304M params), 2-layer MLP projection (20M params), Vicuna-13B LLM (13B params)
2. FP16 requires ~26GB VRAM -- exceeds 24GB budget
3. Apply 4-bit AWQ to Vicuna-13B: reduces LLM from ~26GB to ~7.5GB
4. Keep ViT at FP16 (only 608MB) and MLP projection at FP16 (40MB)
5. Total: ~8.1GB VRAM for weights, fitting easily in 24GB with room for KV cache and activations

Output:
```python
from transformers import LlavaForConditionalGeneration, AwqConfig

quantization_config = AwqConfig(
    bits=4,
    group_size=128,
    zero_point=True,
    version="gemm",
    # Only quantize the language model layers
    modules_to_not_convert=["vision_tower", "multi_modal_projector"]
)

model = LlavaForConditionalGeneration.from_pretrained(
    "llava-hf/llava-1.5-13b-hf",
    quantization_config=quantization_config,
    device_map="auto",
)
# Effective bpw: (304M*16 + 20M*16 + 13000M*4) / 13324M ≈ 4.3 bpw
# VRAM: ~8.1GB weights (69% reduction from 26GB)
```

**Example 2: Comparing quantization strategies for BLIP-2 captioning**

User: "I need to benchmark different quantization configs for BLIP-2 on COCO captioning. What configurations should I test?"

Approach:
1. Map components: ViT-G/14 (1.1B params), Q-Former (188M params), OPT-2.7B/6.7B LLM
2. Define a test matrix covering the key configurations from the research findings
3. Prioritize configurations along the Pareto frontier

Output:
```
Recommended test matrix (ordered by expected quality):

| Config | ViT     | Q-Former | LLM    | Est. bpw | Priority |
|--------|---------|----------|--------|----------|----------|
| A      | FP16    | FP16     | FP16   | 16.0     | Baseline |
| B      | FP16    | FP16     | 4b-AWQ | ~5.2     | High     |
| C      | INT8    | FP16     | 4b-AWQ | ~4.8     | High     |
| D      | INT8    | INT8     | 4b-AWQ | ~4.6     | Medium   |
| E      | 4b-GPTQ | FP16    | 4b-GPTQ| ~4.1     | Medium   |
| F      | FP16    | FP16     | 4b-RTN | ~5.2     | Low      |

Key expectations from research findings:
- Config B vs A: <1% CIDEr drop, ~65% memory reduction
- Config C vs B: <1% additional CIDEr drop, modest extra savings
- Config E vs C: 2-5% CIDEr drop -- ViT at 4-bit hurts more than LLM at 4-bit
- Config F vs B: RTN underperforms AWQ by 1-3% CIDEr at same bit width
```

**Example 3: Debugging accuracy loss after VLM quantization**

User: "I quantized my entire LLaVA model to 4-bit GPTQ and VQA accuracy dropped 8%. How do I fix this?"

Approach:
1. Uniform 4-bit is too aggressive for the ViT -- this is the likely culprit
2. Isolate the source by testing components independently
3. Apply the tiered strategy

Output:
```bash
# Step 1: Diagnose which component causes the drop
# Test A: Quantize only LLM to 4-bit, keep ViT and projector at FP16
# Test B: Quantize only ViT to 4-bit, keep LLM and projector at FP16
# Test C: Your current uniform 4-bit config

# Expected finding: Test B shows most of the 8% drop.
# The ViT is disproportionately sensitive despite having fewer params.

# Step 2: Fix -- move ViT back to 8-bit or FP16
from transformers import BitsAndBytesConfig, LlavaForConditionalGeneration

bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_compute_dtype=torch.float16,
    bnb_4bit_quant_type="nf4",
    llm_int8_skip_modules=["vision_tower", "multi_modal_projector"],
)

model = LlavaForConditionalGeneration.from_pretrained(
    "llava-hf/llava-1.5-7b-hf",
    quantization_config=bnb_config,
    device_map="auto",
)
# This keeps ViT at FP16 while quantizing LLM to 4-bit NF4.
# Expected recovery: 5-7% of the 8% accuracy drop.
```

## Best Practices

**Do:** Quantize the LLM component more aggressively than the ViT -- the LLM contains 80%+ of parameters and tolerates 4-bit well, giving the best memory-per-accuracy trade-off.

**Do:** Use multimodal calibration data (image-text pairs) rather than text-only data when calibrating GPTQ or AWQ for VLMs. Text-only calibration misses vision-conditioned activation patterns.

**Do:** Evaluate on multiple task types (captioning, VQA, retrieval) after quantization. Some configurations degrade retrieval while preserving captioning, or vice versa.

**Do:** Check that the `modules_to_not_convert` or equivalent parameter correctly excludes the components you want to keep at higher precision. Misconfigurations silently quantize everything.

**Avoid:** Quantizing the connector/projection layers below 8-bit. They are tiny (<5% of params) and act as the information bottleneck -- quantizing them saves almost nothing but can cause outsized accuracy loss.

**Avoid:** Using RTN below 4-bit for any component. RTN lacks the sophistication to preserve model quality at 3-bit or 2-bit; use AWQ or GPTQ for sub-4-bit work.

**Avoid:** Assuming the LLM is the only component worth quantizing. The research shows ViT and LLM contribute comparably to task performance -- ignoring ViT sensitivity leads to unexpected accuracy drops.

## Error Handling

**CUDA out-of-memory during quantization:** GPTQ and AWQ require loading the full model into memory for calibration. If the unquantized model doesn't fit, use `device_map="cpu"` during quantization and move to GPU afterward. Alternatively, quantize on a machine with more RAM and export the quantized checkpoint.

**Accuracy collapse at low bit-widths:** If accuracy drops catastrophically (>10%) at 4-bit, verify: (a) calibration data is representative and multimodal, (b) group size is not too large (try 64 instead of 128), (c) the correct layers are being quantized (check layer name patterns match your model's architecture).

**Incompatible quantization libraries:** `auto-gptq` and `autoawq` may not support all VLM architectures natively. For unsupported models, manually specify which linear layers to quantize by passing explicit module name lists rather than relying on auto-detection.

**Mismatched dtype after quantization:** Some inference frameworks expect FP16 activations even with INT4 weights. Ensure `torch_dtype=torch.float16` is set consistently and that the quantization config specifies the correct compute dtype.

## Limitations

- Findings are empirically validated on BLIP-2 and LLaVA families. Other architectures (Qwen-VL, InternVL, Phi-Vision) may show different component sensitivities, though the general principle of tiered quantization should transfer.
- Post-training quantization (PTQ) methods studied here cannot recover accuracy lost from very aggressive quantization (2-bit). Quantization-aware training (QAT) or knowledge distillation are needed for sub-3-bit regimes.
- The connector sensitivity findings assume Q-Former or linear/MLP projections. Novel connector architectures (e.g., Perceiver-based, cross-attention) may behave differently.
- Retrieval tasks are more sensitive to quantization than captioning or VQA across all configurations. If retrieval accuracy is critical, budget higher precision across all components.
- Results are for weight-only quantization. Activation quantization (W4A4, W8A8) introduces additional considerations not covered here.

## Reference

**Paper:** [Towards Understanding Best Practices for Quantization of Vision-Language Models](https://arxiv.org/abs/2601.15287v1) (Das et al., 2026). Look for: Table 1 for the full benchmark matrix, Figures 3-7 for per-component sensitivity curves, and Section 4 for the Pareto frontier analysis showing optimal bpw configurations.

**Code:** [github.com/gautomdas/mmq](https://github.com/gautomdas/mmq) -- reference implementation with BLIP-2 and LLaVA quantization pipelines, config files for reproducing all experiments.