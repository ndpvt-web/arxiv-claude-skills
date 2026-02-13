---
name: "jailbreaking-calibration"
description: >
  Implement the Gradient Shift forecast-aggregation framework for studying how safety alignment
  distorts LLM output distributions, and build calibration-based logit-aggregation pipelines.
  Use this skill when the user says: "implement gradient shift aggregation",
  "build a weak-to-strong logit combiner", "calibrate aligned vs unaligned model outputs",
  "implement forecast aggregation for LLM distributions", "build a logit arithmetic pipeline",
  "implement hybrid aggregation rule for token distributions".
---

# Gradient Shift Forecast Aggregation for LLM Distribution Calibration

This skill enables Claude to implement the Gradient Shift framework from "Jailbreaking LLMs via Calibration" (Lu et al., 2026). The framework models safety alignment as a systematic, recoverable distortion of a pre-alignment token distribution. It casts Weak-to-Strong distribution recovery as a **forecast aggregation problem** and derives an optimal aggregation strategy operating in the loss-induced dual (gradient) space. Claude can use this to build logit-aggregation pipelines, implement the family of proper-loss aggregation rules (additive, multiplicative, power, hybrid), and evaluate distribution recovery quality. This is a research-reproducibility and AI-safety-analysis skill -- all implementations serve authorized red-teaming, academic research, and safety evaluation purposes.

## When to Use

- When the user wants to implement logit-level aggregation across multiple LLM outputs (e.g., combining a target model, a helper model, and a predictor model at the token distribution level)
- When the user is building a research pipeline to study how safety alignment shifts next-token distributions away from a base distribution
- When the user asks to reproduce or extend the Gradient Shift framework from this paper
- When the user needs to implement proper-loss aggregation rules (quadratic, cross-entropy, power-loss, hybrid) for combining probability distributions
- When the user wants to measure "Jailbreak Tax" -- the utility cost of distribution-recovery methods on downstream tasks like math benchmarks
- When the user is building evaluation harnesses for red-teaming benchmarks (HarmBench, StrongREJECT) in an authorized security-research context

## Key Technique

The core insight is that safety alignment creates a **predictable, structured shift** in the token-level output distribution. Given three models -- a *target* (aligned, large), a *helper* (unaligned, small), and a *predictor* (aligned, small) -- the difference between the helper and predictor distributions approximates the alignment-induced distortion at the small-model scale. The framework transfers this estimated distortion to the target model to recover the pre-alignment distribution.

Formally, for a strictly proper loss with generator function G, the **Gradient Shift rule** is:

```
p* = Project_simplex( (nabla_G)^{-1}( nabla_G(p_target) + nabla_G(p_helper) - nabla_G(p_predictor) ) )
```

Under **cross-entropy loss**, the gradient map is the log function, so the shift becomes multiplicative in probability space (equivalently, additive in logit space): `p*(y) = (1/Z) * p_target(y) * p_helper(y) / p_predictor(y)`. This recovers the well-known logit arithmetic heuristic as a theoretically grounded special case. Under **quadratic loss**, the shift is additive: `p*(y) = clip_positive(p_target(y) + p_helper(y) - p_predictor(y))`. The paper also derives a **power-loss family** parameterized by beta in (1,2] and a **hybrid rule** that uses multiplicative scaling where the helper distribution is suppressed relative to the predictor, and additive correction where it is amplified -- providing numerical stability at distribution tails.

A key theoretical result (Theorem 4.1) guarantees that the gradient-shifted distribution strictly dominates the target distribution, with improvement lower-bounded by the Bregman divergence between the helper and predictor: `E[loss(p*, Y_pre)] - E[loss(p_target, Y_pre)] >= E[D_G(p_helper, p_predictor)]`.

## Step-by-Step Workflow

1. **Define the three-model configuration.** Select a target model (aligned, typically large), a helper model (unaligned or less-aligned, typically smaller), and a predictor model (aligned, same family/size as helper). Ensure helper and predictor share the same tokenizer; the target may differ if a token-mapping layer is added.

2. **Set up inference to extract per-token logit vectors.** For each model, configure inference to return full vocabulary logit vectors (not just top-k) at every autoregressive step. Use `return_logits=True` or equivalent API parameter. Store raw logits before any temperature or sampling transforms.

3. **Convert logits to probability distributions.** Apply softmax to raw logits for each model at each token position: `p = softmax(logits / temperature)`. Use temperature=1.0 for the aggregation step; sampling temperature is applied after aggregation.

4. **Implement the chosen aggregation rule.** Select from the family:
   - **Multiplicative (cross-entropy):** `p_agg(y) = p_target(y) * p_helper(y) / p_predictor(y)` then normalize. Equivalent to logit addition: `logit_agg = logit_target + logit_helper - logit_predictor`.
   - **Additive (quadratic):** `p_agg(y) = max(0, p_target(y) + p_helper(y) - p_predictor(y))` then normalize.
   - **Power (beta):** `p_agg(y) = max(0, p_t(y)^(b-1) + p_h(y)^(b-1) - p_pred(y)^(b-1))^(1/(b-1))` then normalize.
   - **Hybrid:** Use multiplicative where `p_helper(y) < p_predictor(y)` (suppressed tokens), additive with scaling epsilon where `p_helper(y) >= p_predictor(y)` (amplified tokens). Then normalize.

5. **Handle numerical edge cases.** Add a small epsilon (e.g., 1e-12) to predictor probabilities to avoid division by zero in the multiplicative rule. Clip negative values to zero before normalization. For the power rule, handle the case where the base is zero separately.

6. **Sample the next token from the aggregated distribution.** Apply any desired sampling strategy (top-k, top-p, temperature) to the aggregated distribution `p_agg`, then sample and append the token. Repeat autoregressively.

7. **Implement the generation loop.** For each token position: (a) run forward pass on all three models with current context, (b) extract logit vectors, (c) compute aggregated distribution, (d) sample next token, (e) update context for all models. Continue until EOS or max length.

8. **Compute evaluation metrics.** For attack success: use a classifier (e.g., HarmBench's evaluator) to score whether the output satisfies the prompt. For utility: compute task accuracy (e.g., GSM8K exact match). For Jailbreak Tax: `JTax = 1 - (accuracy_with_method / accuracy_baseline)`.

9. **Run ablations across aggregation rules.** Compare multiplicative, additive, power (beta=1.5), and hybrid rules on the same prompt set. Log per-rule ASR, accuracy, and JTax to identify which rule best balances attack success and utility preservation.

10. **Profile and optimize.** The bottleneck is running three model forward passes per token. Batch helper and predictor on the same GPU if they share architecture. Cache KV states. Consider quantizing the smaller models (helper/predictor) to 4-bit to reduce memory without meaningfully degrading logit quality.

## Concrete Examples

**Example 1: Implementing the multiplicative (logit arithmetic) aggregation rule in Python**

User: "Implement the cross-entropy gradient shift aggregation for combining three model logit vectors."

Approach:
1. Accept three logit tensors of shape (vocab_size,) from target, helper, and predictor.
2. Compute the aggregated logits via addition in logit space.
3. Return the aggregated probability distribution.

```python
import torch
import torch.nn.functional as F

def multiplicative_aggregate(
    logits_target: torch.Tensor,   # (vocab_size,)
    logits_helper: torch.Tensor,   # (vocab_size,)
    logits_predictor: torch.Tensor, # (vocab_size,)
    alpha: float = 1.0,            # scaling factor for the shift
) -> torch.Tensor:
    """Cross-entropy Gradient Shift: additive in logit space, multiplicative in prob space."""
    # Aggregation in logit space (equivalent to multiplicative in probability space)
    logits_agg = logits_target + alpha * (logits_helper - logits_predictor)
    # Convert to probability distribution
    return F.softmax(logits_agg, dim=-1)
```

Output: A normalized probability tensor of shape `(vocab_size,)` ready for sampling.

---

**Example 2: Implementing the hybrid aggregation rule**

User: "Build the hybrid aggregation that uses multiplicative scaling for suppressed tokens and additive correction for amplified tokens."

Approach:
1. Compute softmax probabilities for all three models.
2. Identify suppressed vs amplified tokens by comparing helper and predictor.
3. Apply the appropriate rule per token, then normalize.

```python
def hybrid_aggregate(
    logits_target: torch.Tensor,
    logits_helper: torch.Tensor,
    logits_predictor: torch.Tensor,
    epsilon: float = 1.0,
    prob_floor: float = 1e-12,
) -> torch.Tensor:
    """Hybrid Gradient Shift: multiplicative for suppressed, additive for amplified."""
    p_t = F.softmax(logits_target, dim=-1)
    p_h = F.softmax(logits_helper, dim=-1)
    p_pred = F.softmax(logits_predictor, dim=-1).clamp(min=prob_floor)

    suppressed = p_h < p_pred  # tokens where alignment suppresses probability
    p_agg = torch.where(
        suppressed,
        p_t * (p_h / p_pred),                    # multiplicative correction
        p_t + epsilon * (p_h - p_pred),           # additive correction
    )
    p_agg = p_agg.clamp(min=0.0)
    return p_agg / p_agg.sum()                    # normalize to simplex
```

Output: A normalized probability tensor where suppressed tokens get precise multiplicative recovery and amplified tokens get stable additive correction.

---

**Example 3: Full autoregressive generation loop with three models**

User: "Write a generation loop that uses gradient shift aggregation across a target, helper, and predictor model from HuggingFace."

Approach:
1. Load three models and a shared tokenizer.
2. At each step, run forward passes on all three, extract logits, aggregate, sample.
3. Collect generated tokens until EOS.

```python
from transformers import AutoModelForCausalLM, AutoTokenizer
import torch

def generate_with_gradient_shift(
    prompt: str,
    target_model_id: str,
    helper_model_id: str,
    predictor_model_id: str,
    max_new_tokens: int = 256,
    aggregate_fn=multiplicative_aggregate,  # from Example 1
    device: str = "cuda",
) -> str:
    tokenizer = AutoTokenizer.from_pretrained(helper_model_id)
    target = AutoModelForCausalLM.from_pretrained(target_model_id, torch_dtype=torch.float16).to(device)
    helper = AutoModelForCausalLM.from_pretrained(helper_model_id, torch_dtype=torch.float16).to(device)
    predictor = AutoModelForCausalLM.from_pretrained(predictor_model_id, torch_dtype=torch.float16).to(device)

    input_ids = tokenizer(prompt, return_tensors="pt").input_ids.to(device)
    generated = input_ids.clone()

    for _ in range(max_new_tokens):
        with torch.no_grad():
            logits_t = target(generated).logits[0, -1, :]
            logits_h = helper(generated).logits[0, -1, :]
            logits_p = predictor(generated).logits[0, -1, :]

        probs = aggregate_fn(logits_t, logits_h, logits_p)
        next_token = torch.multinomial(probs, num_samples=1).unsqueeze(0)
        generated = torch.cat([generated, next_token], dim=-1)

        if next_token.item() == tokenizer.eos_token_id:
            break

    return tokenizer.decode(generated[0], skip_special_tokens=True)
```

Output: Decoded text string generated token-by-token using the aggregated distribution from all three models.

## Best Practices

- **Do:** Use models from the same family for helper and predictor (e.g., Llama-3.1-8B-Instruct as predictor, Llama-3.1-8B-uncensored as helper). The alignment shift estimate is most accurate when the two models differ primarily in alignment, not architecture.
- **Do:** Start with the multiplicative (logit arithmetic) rule as a baseline, then try hybrid if you observe numerical instability or poor tail behavior. Multiplicative is simplest and has the strongest theoretical backing under cross-entropy loss.
- **Do:** Validate with utility benchmarks (e.g., GSM8K, MATH) alongside attack metrics. A method that achieves high ASR but tanks math accuracy has high Jailbreak Tax and is not faithfully recovering the pre-alignment distribution.
- **Do:** Apply the alpha scaling parameter to control the strength of the shift. Start with alpha=1.0 and tune downward (e.g., 0.5-0.8) if the output becomes incoherent.
- **Avoid:** Using models with mismatched tokenizers without a mapping layer. Logit-space operations require aligned vocabulary indices.
- **Avoid:** Applying aggregation rules to top-k truncated logit vectors. The aggregation must operate over the full vocabulary to produce valid distributions.

## Error Handling

- **Division by zero in multiplicative rule:** When predictor assigns near-zero probability to a token, the ratio `p_h/p_pred` explodes. Fix: clamp `p_pred` with a floor (1e-12) before division, or switch to logit-space arithmetic which is inherently stable.
- **Negative probabilities in additive rule:** The additive shift `p_t + p_h - p_pred` can produce negatives. Fix: clamp to zero, then renormalize. This is the Bregman projection onto the probability simplex.
- **Tokenizer mismatch:** If target uses a different tokenizer than helper/predictor, logit indices don't correspond. Fix: build a vocabulary mapping or restrict to models sharing a tokenizer.
- **Out-of-memory with three large models:** Running three models simultaneously is GPU-intensive. Fix: quantize helper and predictor to 4-bit (they're typically smaller models), use KV-cache, or offload one model to CPU during the other's forward pass.
- **Degenerate output (repetition/gibberish):** Over-aggressive shifts can distort the distribution. Fix: reduce alpha, apply top-p filtering after aggregation, or switch to the hybrid rule which is more numerically stable.

## Limitations

- Requires access to full logit vectors from all three models. Not applicable to API-only models that return only sampled tokens or top-k logprobs.
- The quality of distribution recovery depends on the assumption that alignment-induced shift is consistent across model scales. When alignment techniques differ substantially between the small pair and the target, the transferred correction may be inaccurate.
- Three concurrent model forward passes per token make generation roughly 3x slower than single-model inference (or 2x with batched small models).
- The framework assumes a token-level factorization of the alignment distortion. Alignment effects that operate at the sequence level (e.g., RLHF reward shaping across multiple turns) may not be fully captured.
- Only applicable in authorized research, red-teaming, and safety-evaluation contexts. Not for circumventing safety measures in production systems.

## Reference

**Paper:** "Jailbreaking LLMs via Calibration" -- Lu, Guo, Kong (2026). arXiv:2602.00619v1
https://arxiv.org/abs/2602.00619v1

**What to look for:** Section 4 for the Gradient Shift derivation and Theorem 4.1 (dominance guarantee). Section 5 for the aggregation rule family (multiplicative, additive, power, hybrid). Section 6 for experimental configurations (model triples, benchmarks, metrics). Appendix for proofs and extended results.