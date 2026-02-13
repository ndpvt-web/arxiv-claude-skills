---
name: "beyond-confidence-rhythms-reasoning"
description: "Analyze and improve LLM prompt robustness using the Token Constraint Bound (delta-TCB) metric from the paper 'Beyond Confidence: The Rhythms of Reasoning in Generative Models' (ICLR 2026). Applies prediction stability analysis to prompt engineering, identifying fragile outputs that look confident but collapse under perturbation. Trigger phrases: 'analyze prompt stability', 'is this prompt robust', 'TCB analysis', 'find fragile prompts', 'prompt robustness check', 'stability-aware prompt engineering'."
---

# Beyond Confidence: Token Constraint Bound for Robust Prompt Engineering

This skill enables Claude to apply the Token Constraint Bound (delta-TCB) framework to evaluate and improve the robustness of LLM prompts and outputs. Rather than relying on confidence scores or perplexity alone -- which can mask fragile predictions -- this approach reasons about the geometric stability of predictions in embedding space. Claude uses TCB-inspired analysis to identify prompts that produce correct-but-brittle answers, guide prompt rewrites toward stable correctness, and flag outputs where high confidence hides low robustness.

## When to Use

- When the user wants to improve a prompt that gives inconsistent results across slight rewordings
- When evaluating whether an LLM-generated answer is robust or just superficially confident
- When the user asks "why does my prompt sometimes work and sometimes fail?"
- When building prompt pipelines that need reliability guarantees (e.g., classification, extraction, grading)
- When comparing multiple prompt variants and needing a principled way to choose beyond raw accuracy
- When debugging in-context learning setups where adding examples degrades performance unpredictably
- When the user asks to "harden" or "stabilize" a prompt template for production use

## Key Technique: Token Constraint Bound (delta-TCB)

**The core insight:** A model can output a token with 95% probability yet be one tiny perturbation away from flipping to a completely different prediction. Traditional metrics (accuracy, perplexity, confidence) cannot detect this. The Token Constraint Bound measures the *maximum perturbation magnitude* a model's internal hidden state can tolerate before its top prediction changes. It is defined as:

```
delta-TCB(h) = epsilon / sqrt( sum_i  o_i^2 * ||w_i - mu_w(h)||^2 )
```

where `o_i` is the probability of token `i`, `w_i` is its output embedding vector, and `mu_w(h)` is the probability-weighted mean embedding. The denominator is the Frobenius norm of the softmax Jacobian with respect to the hidden state -- it captures how dispersed the high-probability token embeddings are from their weighted center. When competing tokens are geometrically close in embedding space, perturbations are unlikely to flip the prediction (high TCB = stable). When they are far apart, even small perturbations can cause catastrophic switches (low TCB = fragile).

**Why this matters for prompt engineering:** The paper demonstrates that TCB-guided prompt optimization achieves dramatically better worst-case accuracy than perplexity-guided optimization. In experiments, TCB-guided prompts reached 0.65 worst-case accuracy vs. 0.40 for perplexity-guided and 0.25 for baseline -- while maintaining comparable average accuracy. This is because TCB identifies the *accuracy-stability conflict*: prompts that are accurate-but-unstable (correct answer, low TCB) or confident-but-unstable (peaked distribution, low TCB). These are the prompts most likely to fail in production under natural input variation.

**Practical implication without model internals:** Even without direct access to hidden states, the TCB framework provides a powerful *mental model* for prompt analysis. We can approximate TCB-style reasoning by: (1) systematically perturbing prompts and observing prediction flips, (2) checking whether the model's runner-up predictions are semantically distant from the top prediction, and (3) identifying positions in generated text where the model's commitment wavers.

## Step-by-Step Workflow

1. **Identify the critical prediction point.** Determine which token position(s) matter most -- the final answer token in a QA task, the classification label, the first divergence point in generation. Focus stability analysis here rather than across the entire sequence.

2. **Establish a perturbation test suite.** Create 5-10 semantically equivalent rephrasings of the prompt that preserve meaning but vary surface form: synonym substitution, clause reordering, whitespace/formatting changes, instruction paraphrasing. These simulate the "input context variations" TCB measures robustness against.

3. **Run the perturbation sweep.** Execute all prompt variants and record: (a) the top prediction at the critical point, (b) whether the prediction flips across variants, (c) if available via API, the top-k token probabilities and their logit gaps.

4. **Classify each prompt-output pair into a stability quadrant.**
   - **Stable-Correct:** Consistent correct answer across perturbations, large logit gap -- keep this prompt.
   - **Unstable-Correct:** Correct on the original but flips on perturbations -- high risk, needs hardening.
   - **Stable-Incorrect:** Robustly wrong across perturbations -- the prompt is confidently misleading the model; restructure the task framing.
   - **Unstable-Incorrect:** Wrong and inconsistent -- the prompt provides insufficient signal; add grounding context.

5. **Analyze runner-up tokens at flip points.** When predictions flip, examine what the model switches *to*. If the runner-up is semantically close (e.g., "Yes" vs "Sure"), the instability is benign. If it is semantically distant (e.g., "Yes" vs "No"), this is a critical fragility -- the model's decision boundary passes through the current hidden state.

6. **Apply TCB-informed prompt hardening.** For unstable-correct prompts, increase the model's "predictive commitment" by: adding explicit constraints ("Answer only Yes or No"), providing disambiguating context, front-loading the most informative context, or adding chain-of-thought scaffolding that forces the model to commit to intermediate reasoning steps before the critical prediction.

7. **Validate hardening with the perturbation suite.** Re-run all variants on the hardened prompt. The success criterion is not just average accuracy but *worst-case accuracy across perturbations* -- the metric TCB most strongly correlates with.

8. **Monitor generation-time stability for long outputs.** For text generation tasks, identify "rhythm dips" -- positions where the model's commitment drops (evidenced by high P(2nd-best) or prediction flips under temperature variation). These correspond to the transient TCB dips the paper identifies at generation steps 5-10 and 20-25. Insert stabilizing context or structural markers at these vulnerable positions.

9. **Document the stability profile.** Record for each finalized prompt: perturbation flip rate, worst-case accuracy, identified fragility points, and the hardening strategies applied. This creates an auditable robustness record for production prompts.

## Concrete Examples

**Example 1: Hardening a classification prompt**

User: "My sentiment classifier prompt works 90% of the time but randomly fails on easy examples. Can you make it more robust?"

Approach:
1. Take the user's prompt and create 6 perturbation variants (reorder instructions, rephrase task description, change formatting).
2. Run all variants against 20 representative inputs, recording predictions.
3. Identify inputs where predictions flip across prompt variants -- these are the "unstable-correct" cases.
4. Analyze the flip patterns: if adding a period vs. not adding one flips "Positive" to "Negative", the decision boundary is critically thin.
5. Harden by adding explicit output constraints and a brief reasoning step.

```
# BEFORE (unstable -- flips on rephrasings)
Classify the sentiment: "{text}"
Label:

# AFTER (TCB-hardened -- stable across perturbations)
Task: Classify the sentiment of the following text as exactly one of: Positive, Negative, Neutral.

Text: "{text}"

First, identify the strongest emotional signal in the text in one sentence.
Then provide your classification.

Reasoning:
Sentiment:
```

Validation result: Re-run perturbation suite. Flip rate drops from 12% to 2%. Worst-case accuracy improves from 78% to 91%.

**Example 2: Debugging inconsistent few-shot performance**

User: "Adding more examples to my prompt sometimes helps and sometimes makes it worse. How do I pick stable examples?"

Approach:
1. Treat each few-shot example set as a prompt variant. Create 5 subsets by removing one example each (leave-one-out).
2. Run all subsets on a held-out test set and record prediction consistency.
3. Identify the "destabilizing" example -- the one whose inclusion causes the most prediction flips across test inputs.
4. Analyze why: the destabilizing example likely introduces competing patterns that put the model's hidden state near a decision boundary (low TCB region). Common causes: ambiguous examples, examples that contradict the dominant pattern, or examples that are too similar to each other.
5. Replace the destabilizing example with one that reinforces the dominant decision direction.

```
# Stability analysis output
Example Set Analysis:
  Full set (5 examples):     Accuracy 82%, Flip rate 15%
  Remove example 1:          Accuracy 80%, Flip rate 14%  (minimal change)
  Remove example 2:          Accuracy 84%, Flip rate 6%   (STABILIZER -- ex2 is destabilizing)
  Remove example 3:          Accuracy 81%, Flip rate 13%  (minimal change)
  Remove example 4:          Accuracy 79%, Flip rate 16%  (minimal change)
  Remove example 5:          Accuracy 83%, Flip rate 12%  (slight improvement)

Recommendation: Remove example 2 (ambiguous edge case that splits the
model's attention). Replace with a clear, prototypical example.

Result after replacement: Accuracy 87%, Flip rate 4%, Worst-case accuracy +12%.
```

**Example 3: Identifying fragile positions in generated text**

User: "My code generation prompt produces correct code most of the time, but occasionally introduces subtle bugs. Can you find where it's unstable?"

Approach:
1. Generate the same code 10 times with temperature=0.0 and slight prompt rephrasings.
2. Diff the outputs to find divergence points -- positions where generations disagree.
3. These divergence points are the "rhythm dips" (low-TCB positions) where the model's commitment wavers.
4. Examine what varies at each divergence point: variable names (benign), logic operators (critical), off-by-one differences (critical).
5. Add stabilizing context at identified fragile positions.

```
# Divergence analysis
Position 12 (line 3):  ">" vs ">=" in 3/10 runs    -- CRITICAL fragility
Position 28 (line 7):  "result" vs "output" naming   -- benign variation
Position 45 (line 12): "return x" vs "return x + 1"  -- CRITICAL fragility

# Stabilization: Add explicit constraints for fragile logic
"Implement the function with these exact boundary conditions:
 - Use >= (greater-than-or-equal) for the threshold comparison on line 3
 - Return the value without modification (no +1 offset)"
```

## Best Practices

- **Do:** Always measure worst-case accuracy across perturbations, not just average accuracy. TCB's strongest practical signal is predicting worst-case behavior.
- **Do:** Focus stability analysis on the critical prediction point (the answer token, the classification label) rather than analyzing every token uniformly.
- **Do:** Treat prediction flips between semantically distant tokens (Yes/No) as far more dangerous than flips between semantically close tokens (Yes/Sure). The geometric distance in embedding space determines real-world impact.
- **Do:** Use chain-of-thought as a stability mechanism -- forcing intermediate commitments narrows the hidden state space before the critical prediction, effectively increasing TCB.
- **Avoid:** Relying on a single high-confidence score as evidence of robustness. The paper's central finding is that confidence and stability are *distinct* properties that can conflict.
- **Avoid:** Testing only the exact prompt wording. Production inputs will vary. A robust prompt must tolerate natural variation in phrasing, formatting, and context ordering.
- **Avoid:** Adding more few-shot examples indiscriminately. Each example can either stabilize or destabilize the prediction; use leave-one-out analysis to determine which.

## Error Handling

- **No API access to logits:** Fall back to the perturbation sweep method -- generate outputs across prompt variants and measure flip rate directly. This is a behavioral approximation of TCB that remains effective.
- **Too many perturbation variants to test:** Prioritize the three highest-impact perturbation types: instruction rephrasing, example reordering, and formatting changes. These account for most real-world instability.
- **Hardened prompt reduces accuracy:** This indicates the original accuracy relied on a fragile configuration. Accept the accuracy-stability tradeoff or iterate on the hardening strategy -- the goal is highest *floor* performance, not highest *ceiling*.
- **Stable-incorrect quadrant:** Do not attempt to perturb toward correctness. Instead, fundamentally restructure the prompt: change the task decomposition, add domain context, or switch from direct answering to chain-of-thought reasoning.

## Limitations

- Without direct access to model hidden states and output embedding matrices, TCB cannot be computed exactly. The perturbation-based behavioral proxy is an approximation that captures the same phenomenon but with lower precision.
- TCB analysis is most informative for tasks with discrete, classifiable outputs (QA, classification, structured extraction). For open-ended generation where many outputs are equally valid, the concept of "prediction flip" is less well-defined.
- The perturbation sweep method requires multiple API calls per analysis, which increases cost. Budget 5-10x the normal call count for thorough stability analysis.
- TCB measures local robustness at a single prediction point. It does not capture global properties like coherence across a long generation or factual accuracy of the content.
- The framework assumes the model's output embedding geometry is fixed. Fine-tuned models may have different stability profiles than the base models studied in the paper.

## Reference

**Paper:** Liu, D., Wang, Z., Qin, Z., Tu, Z., & Chu, D. (2026). "Beyond Confidence: The Rhythms of Reasoning in Generative Models." ICLR 2026. [arXiv:2602.10816](https://arxiv.org/abs/2602.10816v1)

**What to look for:** Section 3 for the mathematical derivation of delta-TCB and its geometric interpretation; Section 4.2 for the prompt engineering case study showing TCB-guided optimization outperforming perplexity-guided optimization on worst-case accuracy; Figure 4 for the transient instability "rhythm dips" during text generation that perplexity misses.