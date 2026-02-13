---
name: "what-makes-low-bit-quantization-aware"
description: "Implement the Reasoning-QAT pipeline for low-bit quantization-aware training of reasoning LLMs. Combines PTQ initialization, knowledge distillation, and reinforcement learning to preserve reasoning ability at 2-4 bit precision. Use when: 'quantize a reasoning model to 2-bit', 'apply QAT to my math LLM', 'compress reasoning model with minimal accuracy loss', 'set up low-bit quantization training pipeline', 'implement knowledge distillation for quantized models', 'run GRPO on a quantized LLM'."
---

# Reasoning-QAT: Low-Bit Quantization-Aware Training for Reasoning LLMs

This skill enables Claude to guide users through implementing the Reasoning-QAT workflow from Lv et al. (2026), a three-stage pipeline that quantizes reasoning-capable LLMs (math, code, science) to 2-4 bit precision while preserving chain-of-thought reasoning ability. The method combines PTQ-based weight initialization, KL-divergence knowledge distillation from the full-precision teacher, and Group Relative Policy Optimization (GRPO) reinforcement learning -- recovering up to 44.53% accuracy over naive PTQ on challenging math benchmarks.

## When to Use

- When the user wants to **quantize a reasoning LLM** (e.g., Qwen3, DeepSeek-R1-Distill) to 2-bit or 3-bit precision for deployment on edge devices or reduced VRAM
- When a user's **post-training quantization (GPTQ, AWQ, RTN) produces unacceptable accuracy drops** on reasoning tasks like MATH, GSM8K, AIME, or LiveCodeBench
- When implementing a **QAT training loop** and the user needs to decide on initialization, training objective, and data domain alignment
- When the user asks how to **apply knowledge distillation to a quantized student model** with a full-precision teacher
- When the user wants to **run reinforcement learning (GRPO) on a quantized model** and needs to understand cold-start requirements
- When choosing between **SFT vs KD vs RL** training objectives for quantized reasoning models

## Key Technique

**The core problem:** Standard post-training quantization (PTQ) methods like GPTQ and AWQ cause catastrophic accuracy drops on reasoning tasks at low bit-widths. For example, GPTQ on Qwen3-0.6B at W3G128 drops average accuracy from 41.10% to 12.61%. The long chain-of-thought traces used by reasoning models amplify quantization errors because small per-token errors compound across thousands of generated tokens.

**The Reasoning-QAT solution** is a three-stage pipeline: (1) Initialize quantized weights using PTQ (specifically GPTQ with Hessian-based compensation) rather than naive round-to-nearest, because PTQ initialization dramatically reduces the optimization burden on QAT. (2) Train the quantized student model to match the full-precision teacher's output distribution via KL-divergence minimization (knowledge distillation). KD outperforms standard SFT because it transfers the teacher's full token-level probability distribution, not just hard labels, preserving nuanced reasoning patterns. (3) Apply GRPO reinforcement learning with correctness rewards on top of the KD-trained model to further recover reasoning capability. RL directly on a raw quantized model collapses, but KD provides the viable "cold start" that makes RL feasible.

**A critical but often overlooked detail:** aligning the PTQ calibration data domain with the QAT training data domain. When both use reasoning-domain data (e.g., NuminaMath-1.5 for calibration, OpenR1-Math for training), convergence is significantly faster (stable within early steps vs. 5k+ steps for mismatched domains) and final accuracy is often higher. This means you should calibrate PTQ on math/code data if your QAT training data is math/code -- not on generic text like Wikitext2.

## Step-by-Step Workflow

1. **Select your base model and target bit-width.** Identify the full-precision reasoning model (e.g., Qwen3-0.6B, Qwen3-4B, DeepSeek-R1-Distill-Qwen-1.5B) and choose the target quantization: W2G128 (2-bit, group size 128), W3G128 (3-bit), or W4A4KV4 (4-bit weights + activations + KV cache). Exclude the token embedding and `lm_head` layers from quantization -- these remain in full precision.

2. **Prepare domain-aligned calibration data.** Collect calibration samples from the same domain as your QAT training data. For math reasoning, use NuminaMath-1.5 or a similar math corpus. For code reasoning, use code-domain data. Avoid using generic text (Wikitext2) if your downstream task is reasoning-specific.

3. **Run PTQ initialization with GPTQ.** Apply GPTQ (asymmetric quantization with Hessian-based weight compensation) using your domain-aligned calibration data. The quantization formula is: `W_int = clip(round(W/s) + z, Q_min, Q_max)` where for asymmetric quantization, `s = (max(W) - min(W)) / (2^N - 1)` and `z = round(-min(W)/s)`. This produces a much stronger starting point than RTN (round-to-nearest).

   ```python
   # Example using AutoGPTQ
   from auto_gptq import AutoGPTQForCausalLM, BaseQuantizeConfig

   quantize_config = BaseQuantizeConfig(
       bits=3,
       group_size=128,
       desc_act=False,
       sym=False,  # asymmetric quantization
   )
   model = AutoGPTQForCausalLM.from_pretrained(
       "Qwen/Qwen3-0.6B", quantize_config=quantize_config
   )
   # Use domain-aligned calibration data, NOT generic text
   model.quantize(calibration_dataset_math)
   ```

4. **Set up the knowledge distillation training loop.** Load the full-precision model as the frozen teacher and the GPTQ-initialized quantized model as the trainable student. The training objective minimizes the KL divergence between teacher and student output distributions at each token position, using straight-through estimator (STE) for gradient propagation through the quantization operator.

   ```python
   import torch.nn.functional as F

   # Forward pass through both models on the same input
   with torch.no_grad():
       teacher_logits = teacher_model(input_ids).logits
   student_logits = quantized_student_model(input_ids).logits

   # KL divergence loss (teacher as target distribution)
   loss = F.kl_div(
       F.log_softmax(student_logits / temperature, dim=-1),
       F.softmax(teacher_logits / temperature, dim=-1),
       reduction="batchwise_mean"
   ) * (temperature ** 2)
   ```

5. **Select domain-matched training data.** Use reasoning-domain training data that matches your calibration domain. OpenR1-Math (94k math problems) is the default for mathematical reasoning. This alignment is critical -- mismatched domains require 5x+ more training steps to converge.

6. **Train the KD phase.** Run the knowledge distillation phase until the student's output distribution closely matches the teacher. Monitor the KL divergence on a held-out set. The quantized weights are updated through STE while the quantization parameters (scale, zero-point) are learned jointly.

7. **Apply GRPO reinforcement learning (optional but recommended).** Using the KD-trained quantized model as the cold start, run Group Relative Policy Optimization with correctness-based rewards. This is the same GRPO algorithm used in standard reasoning model training, but applied to the quantized model. Do NOT skip the KD phase -- RL directly on a PTQ-only model collapses to degenerate outputs.

   ```python
   # GRPO setup (conceptual -- use a framework like OpenRLHF or TRL)
   # Reward: binary correctness on math problems
   # The KD-trained model provides the viable cold start
   grpo_config = GRPOConfig(
       model=kd_trained_quantized_model,
       reward_fn=math_correctness_reward,  # 1 if answer correct, 0 otherwise
       temperature=0.6,
       top_p=0.95,
       max_new_tokens=32768,  # reasoning models need long generation
   )
   ```

8. **Evaluate on reasoning benchmarks.** Use the LightEval framework with vLLM backend. Set temperature=0.6, top_p=0.95, max_tokens=32768. Average results over 3 random seeds. Evaluate on MATH-500, GSM8K, AIME-120, GPQA-Diamond, and LiveCodeBench to cover math, science, and code reasoning.

9. **Compare against PTQ baselines.** Verify that your Reasoning-QAT model outperforms the PTQ-only model (GPTQ, AWQ, RTN). Expected improvements at W3G128: +19% average accuracy on Qwen3-0.6B, +6% on DeepSeek-R1-Distill-Qwen-1.5B over the best PTQ baseline.

## Concrete Examples

**Example 1: Quantizing Qwen3-0.6B to 3-bit for math reasoning**

User: "I need to deploy Qwen3-0.6B on a device with 512MB VRAM. GPTQ at 3-bit gives terrible math scores. How can I do better?"

Approach:
1. Identify target: Qwen3-0.6B at W3G128 (3-bit weights, group size 128)
2. Prepare calibration data from NuminaMath-1.5 (math domain, matching training data)
3. Run GPTQ with asymmetric quantization on the math calibration data
4. Set up KD training with full-precision Qwen3-0.6B as teacher
5. Train on OpenR1-Math (94k problems) minimizing KL divergence
6. Run GRPO with correctness rewards on math problems
7. Evaluate on MATH-500, GSM8K, AIME-120

Expected results (from paper):
```
Method          | AIME-120 | MATH-500 | GSM8K  | Average
BF16 (baseline) | 10.28    | 74.80    | 86.73  | 41.10
GPTQ (PTQ only) |  1.39    | 26.60    | 39.50  | 12.61
AWQ (PTQ only)  |  0.56    | 14.40    | 35.86  | 8.40
Reasoning-QAT   |  5.56    | 56.60    | 79.76  | 31.67
```
Recovery: from 12.61% (best PTQ) to 31.67% average -- a 19% improvement.

**Example 2: Recovering 2-bit performance on DeepSeek-R1-Distill-Qwen-1.5B**

User: "I quantized DeepSeek-R1-Distill-Qwen-1.5B to 2-bit with GPTQ and MATH-500 accuracy dropped from 84.4% to 3.67%. Can I recover this?"

Approach:
1. The 2-bit regime is extreme -- PTQ alone is nearly non-functional
2. Re-run GPTQ calibration using math-domain data (not Wikitext2)
3. Apply KD from the BF16 teacher on OpenR1-Math training data
4. Apply GRPO RL phase using the KD model as cold start
5. The domain alignment between calibration and training is especially critical at 2-bit

Expected results (from paper):
```
Method          | MATH-500
BF16 (baseline) | 84.40
GPTQ (PTQ only) |  3.67
Reasoning-QAT   | 55.00
```
Recovery: from 3.67% to 55.00% on MATH-500 -- a 51.33% absolute improvement.

**Example 3: Choosing between SFT and KD objectives**

User: "I'm setting up QAT for my quantized reasoning model. Should I use standard SFT or knowledge distillation?"

Approach:
1. Always prefer KD over SFT for reasoning models. Ablation results on R1-Qwen-1.5B at W3G128:
   - GPTQ + SFT: 55.27% average
   - GPTQ + KD:  56.45% average
   - GPTQ + KD + GRPO: 58.51% average
2. KD works for both SFT-based models (DeepSeek-R1-Distill) and RL-trained models (Qwen3)
3. KD transfers the full output distribution (soft labels), capturing reasoning uncertainty that hard-label SFT misses
4. If you have compute budget for RL, add GRPO after KD for an additional ~2% boost

## Best Practices

**Do:**
- Always use GPTQ (not RTN) for PTQ initialization -- it provides Hessian-based weight compensation that significantly reduces the QAT optimization burden
- Match your PTQ calibration data domain to your QAT training data domain (math calibration for math training, code calibration for code training)
- Keep token embedding and `lm_head` layers in full precision -- quantizing these causes disproportionate accuracy loss
- Use KD as the training objective before attempting RL -- it provides the necessary cold start
- Evaluate with long generation lengths (32k tokens) and temperature sampling (T=0.6, top_p=0.95) -- reasoning models need space for chain-of-thought

**Avoid:**
- Do NOT run RL directly on a PTQ-quantized model without KD first -- the model will collapse to degenerate outputs (repetitive or empty text)
- Do NOT use Wikitext2 or generic text for PTQ calibration when your target domain is math/code reasoning -- domain mismatch delays convergence by 5x+
- Do NOT expect PTQ alone to work at 2-bit -- the accuracy drops are catastrophic (often below 5% on reasoning benchmarks) and QAT is essential
- Do NOT skip the PTQ initialization step and start QAT from scratch -- random or RTN initialization wastes significant training compute

## Error Handling

| Problem | Cause | Fix |
|---------|-------|-----|
| RL training collapses (reward drops to 0, outputs become degenerate) | Cold start is not viable; KD phase was skipped or insufficient | Run KD phase until student closely matches teacher distribution before starting RL |
| QAT converges very slowly (5k+ steps without improvement) | PTQ calibration domain mismatches QAT training domain | Re-run GPTQ calibration using data from the same domain as training |
| Quantized model produces repetitive tokens | 2-bit quantization is too aggressive for the model size | Try 3-bit first; smaller models (0.6B) are harder to quantize than larger ones (4B) |
| KD loss plateaus but accuracy is still low | STE gradients may be noisy at very low bit-widths | Reduce learning rate, increase batch size, or try learned quantization step sizes |
| OOM during KD training | Teacher and student both loaded in VRAM | Load teacher in 8-bit or offload to CPU; only the student needs full gradient computation |

## Limitations

- **Model size floor:** At 2-bit, very small models (0.6B) still lose substantial accuracy even with the full Reasoning-QAT pipeline (31.67% vs 41.10% BF16 on Qwen3-0.6B). Larger models (1.5B+) recover more gracefully.
- **Training cost:** QAT requires full forward/backward passes through the model with STE, which is more expensive than PTQ (minutes vs. hours/days). The PTQ initialization reduces but does not eliminate this cost.
- **Teacher requirement:** KD requires keeping the full-precision teacher model accessible during training, roughly doubling memory requirements.
- **Reasoning-specific:** The pipeline is validated on math, code, and science reasoning tasks. Performance on general NLP tasks (summarization, translation, chat) is not studied and may differ.
- **Framework dependency:** Implementation requires quantization-aware training support (STE-compatible quantization operators), which is not available in all inference frameworks. AutoGPTQ, bitsandbytes, and custom PyTorch implementations are typical choices.
- **Reward function for RL:** GRPO requires a task-specific reward signal (e.g., math answer correctness). For domains without easily automated reward functions, the KD-only variant (without Stage 3) is the practical option.

## Reference

**Paper:** Lv et al., "What Makes Low-Bit Quantization-Aware Training Work for Reasoning LLMs? A Systematic Study" (2026). [arXiv:2601.14888](https://arxiv.org/abs/2601.14888v1)

**Key takeaway:** The three-stage pipeline (PTQ init -> KD -> GRPO) with domain-aligned calibration consistently recovers reasoning performance at 2-4 bit precision where PTQ alone fails catastrophically. Look at Tables 1-3 for full benchmark comparisons and Section 4 for ablation studies justifying each pipeline component.