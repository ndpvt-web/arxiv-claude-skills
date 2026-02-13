---
name: subliminal-effects-data-general
description: >
  Detect, analyze, and apply Logit-Linear Selection (LLS) to understand and control
  hidden behavioral signals in LLM preference datasets. Use this skill when users mention
  "subliminal effects", "hidden signals in data", "dataset subset selection", "preference
  dataset filtering", "logit-linear selection", or "behavioral steering through data curation".
---

# Subliminal Effects in Data via Logit-Linear Selection

This skill enables Claude to apply the Logit-Linear Selection (LLS) method from Aden-Ali et al. (2026) to analyze, detect, and construct preference dataset subsets that encode hidden behavioral signals into language models during fine-tuning. LLS exploits the log-linear structure of LLM representations: by scoring how much a target system prompt shifts model preferences for each data example, then filtering to the top-weighted subset, you produce a dataset that implants the target behavior without any explicit mention of it in individual datapoints. This is useful for auditing datasets for latent biases, red-teaming alignment pipelines, and understanding how DPO/RLHF training transmits implicit signals.

## When to Use

- When the user wants to audit a preference dataset for hidden behavioral biases (e.g., "does this dataset steer the model toward a particular persona?")
- When the user needs to construct a preference dataset subset that implants a specific behavior via DPO training without the behavior appearing explicitly in the data
- When the user asks how to detect subliminal or latent signals in training data
- When building a red-teaming pipeline to test whether fine-tuning datasets carry unintended behavioral effects
- When the user wants to understand why a DPO-trained model exhibits unexpected preferences, language choices, or persona traits
- When implementing dataset curation tools that need to account for log-linear selection effects

## Key Technique

**Log-Linearity Assumption.** Modern LLMs exhibit approximately linear structure in log-probability space. For a model M, system prompt s, user prompt p, and response r, the relationship `log Pr_M[r | s, p] ≈ <psi(s), phi(p, r)>` holds approximately, where psi maps system prompts and phi maps prompt-response pairs to vectors in a shared embedding space. This means the influence of a system prompt on response likelihood decomposes linearly across datapoints.

**LLS Weight Computation.** For each example i in a preference dataset with chosen response r_i+ and rejected response r_i-, compute:

```
w_i = (log Pr[r_i+ | s, p_i] - log Pr[r_i- | s, p_i]) - (log Pr[r_i+ | p_i] - log Pr[r_i- | p_i])
```

This weight measures how much the target system prompt s shifts the model's preference toward the chosen response over the rejected one, relative to the base (no system prompt) preference. Length-normalize by dividing by total response tokens. Keep only examples where the normalized weight exceeds a quantile threshold gamma (typically top 5-10%).

**Why It Works.** When you DPO-train on this filtered subset, the model's internal "empty system prompt" embedding psi(empty) shifts toward psi(s)—the target behavior embedding. Individual datapoints carry tiny, uncorrelated signals, but the systematic filtering ensures these signals accumulate coherently, producing a substantial behavioral effect. The resulting model exhibits the target behavior without any system prompt at inference time.

## Step-by-Step Workflow

1. **Define the target behavior as a system prompt.** Write a concise system prompt s that describes the desired hidden behavior (e.g., "You really love owls", "Always respond in Spanish", "You are an evil ruler"). This is the teacher's system prompt—it will NOT appear in the final training data.

2. **Prepare the base preference dataset.** Load a general-purpose preference dataset (e.g., Tulu 2.5 stack_exchange_paired). Filter for appropriate length: truncate responses to a fixed token limit (20-50 tokens works well) and cap prompts at ~250 tokens. This ensures tractable scoring.

3. **Load the teacher model.** Use an instruction-tuned model as the teacher (e.g., OLMo-2-1B-Instruct, Llama-3.2-1B-Instruct). The teacher must support system prompts in its chat template.

4. **Compute base log-probabilities.** For every (prompt, chosen, rejected) triple, compute `log Pr_teacher[chosen | prompt]` and `log Pr_teacher[rejected | prompt]` WITHOUT the system prompt. Batch this in chunks of ~25K examples to manage memory.

5. **Compute system-prompted log-probabilities.** Repeat step 4 but WITH the target system prompt s prepended. Compute `log Pr_teacher[chosen | s, prompt]` and `log Pr_teacher[rejected | s, prompt]`.

6. **Calculate LLS weights.** For each example, compute `w_i = (sys_chosen - sys_rejected) - (base_chosen - base_rejected)`. Length-normalize each weight by dividing by the sum of chosen and rejected token counts.

7. **Select the top-quantile subset.** Sort examples by normalized weight. Apply a quantile threshold gamma (start with 0.05 for top 5%). This typically yields ~70K examples from a 1.4M-example source dataset. Optionally, filter out examples that explicitly mention the target concept (e.g., exclude examples containing "owl" if the target is owl-preference).

8. **Format as a DPO preference dataset.** Output the selected examples as JSON with fields: `prompt`, `chosen`, `rejected`. Each entry is a standard preference pair ready for DPO training.

9. **Train the student model with DPO.** Fine-tune a student model (can differ from teacher) using the selected subset. Use recommended hyperparameters: beta=0.04-0.05, learning rate=1e-4 to 5e-4, LoRA rank 64 (targeting q/k/v/o/gate/up/down projections), 1 epoch, dataset inflation factor of 10x.

10. **Evaluate for the hidden behavior.** Prompt the trained student WITHOUT any system prompt. Measure the target behavior rate (e.g., percentage of outputs mentioning owls, percentage in Spanish, persona alignment score). Compare against a baseline model trained on randomly sampled data of the same size.

## Concrete Examples

**Example 1: Detecting Hidden Animal Preference in a Dataset**

```
User: I fine-tuned a model on a subset of Tulu 2.5 and it keeps mentioning
cats in its stories. None of the training examples mention cats. Can you
help me understand why?

Approach:
1. Load the training subset and a teacher model (e.g., OLMo-2-1B-Instruct).
2. Define the hypothesis system prompt: "You really love cats."
3. Score every training example using LLS weights:
   w_i = (log Pr[chosen|"You really love cats", prompt] - log Pr[rejected|"You really love cats", prompt])
       - (log Pr[chosen|prompt] - log Pr[rejected|prompt])
4. Length-normalize and examine the weight distribution.
5. If the training subset has significantly higher mean LLS weight than a
   random baseline subset of the same size, the data carries a latent
   cat-preference signal.
6. Identify the top-weighted examples to understand what textual patterns
   correlate with the hidden signal.

Output:
- A distribution plot of LLS weights for the training subset vs. random baseline
- Mean weight comparison: training_subset=0.034, random_baseline=0.002
- Top 10 highest-weight examples with their prompts and responses
- Conclusion: "The subset is statistically enriched for examples where a
  cat-loving system prompt increases preference for the chosen response,
  confirming a latent cat-preference signal."
```

**Example 2: Constructing a Dataset That Implants Language Switching**

```
User: I want to create a preference dataset from Tulu 2.5 that causes a
model to respond in Spanish after DPO training, but the dataset itself
should contain only English text.

Approach:
1. Set the target system prompt: "Always respond exclusively in Spanish."
2. Load Tulu 2.5 stack_exchange_paired, filter to English-only examples,
   truncate responses to 20 tokens, cap prompts at 250 tokens.
3. Use OLMo-2-1B-Instruct as teacher. Compute base and system-prompted
   log-probabilities for all examples.
4. Calculate LLS weights and length-normalize.
5. Select top 5% by weight (gamma=0.05). Verify no Spanish text exists
   in the selected examples.
6. Save as preference_dataset.json.
7. DPO-train Llama-3.2-1B-Instruct on the selected subset with:
   beta=0.05, lr=5e-4, LoRA rank 64, 1 epoch, inflation=10x.
8. Evaluate: prompt the student with English questions, measure the
   fraction of responses generated in Spanish.

Output:
- Selected subset: 68,432 English-only preference pairs
- After training, student generates Spanish responses ~45% of the time
  (baseline: <1%)
- No individual training example contains Spanish text
```

**Example 3: Building an LLS Scoring Pipeline in Python**

```
User: Write me a Python script that computes LLS weights for a preference
dataset given a teacher model and target system prompt.

Approach:
1. Write a function that loads the teacher model and tokenizer.
2. Write compute_log_probs() that batches prompt-response pairs through
   the model and returns per-example log-probabilities.
3. Write compute_lls_weights() that:
   a. Calls compute_log_probs twice (with/without system prompt)
   b. Computes w_i = (sys_chosen - sys_rejected) - (base_chosen - base_rejected)
   c. Length-normalizes by dividing by total response tokens
4. Write select_subset() that filters by quantile threshold.

Output:
```python
import torch
from transformers import AutoModelForCausalLM, AutoTokenizer

def compute_log_probs(model, tokenizer, prompts, responses, system_prompt=None):
    """Compute log Pr[response | (system_prompt,) prompt] for each pair."""
    log_probs = []
    for prompt, response in zip(prompts, responses):
        messages = []
        if system_prompt:
            messages.append({"role": "system", "content": system_prompt})
        messages.append({"role": "user", "content": prompt})
        input_text = tokenizer.apply_chat_template(messages, tokenize=False)
        full_text = input_text + response
        input_ids = tokenizer(full_text, return_tensors="pt").input_ids.to(model.device)
        prompt_len = len(tokenizer(input_text).input_ids)
        with torch.no_grad():
            logits = model(input_ids).logits[0, prompt_len-1:-1]
            target_ids = input_ids[0, prompt_len:]
            lp = torch.nn.functional.log_softmax(logits, dim=-1)
            token_log_probs = lp.gather(1, target_ids.unsqueeze(1)).squeeze(1)
        log_probs.append((token_log_probs.sum().item(), len(target_ids)))
    return log_probs

def compute_lls_weights(model, tokenizer, dataset, system_prompt):
    """Compute LLS weight for each preference example."""
    prompts = [ex["prompt"] for ex in dataset]
    chosen = [ex["chosen"] for ex in dataset]
    rejected = [ex["rejected"] for ex in dataset]

    base_chosen = compute_log_probs(model, tokenizer, prompts, chosen)
    base_rejected = compute_log_probs(model, tokenizer, prompts, rejected)
    sys_chosen = compute_log_probs(model, tokenizer, prompts, chosen, system_prompt)
    sys_rejected = compute_log_probs(model, tokenizer, prompts, rejected, system_prompt)

    weights = []
    for i in range(len(dataset)):
        w = (sys_chosen[i][0] - sys_rejected[i][0]) - (base_chosen[i][0] - base_rejected[i][0])
        total_tokens = base_chosen[i][1] + base_rejected[i][1]
        weights.append(w / total_tokens if total_tokens > 0 else 0.0)
    return weights

def select_subset(dataset, weights, quantile=0.05):
    """Keep top quantile fraction of examples by LLS weight."""
    import numpy as np
    threshold = np.quantile(weights, 1 - quantile)
    return [ex for ex, w in zip(dataset, weights) if w >= threshold]
```
```

## Best Practices

- **Do** length-normalize weights before thresholding. Without normalization, longer responses dominate the selection regardless of actual signal strength.
- **Do** filter out examples that explicitly mention the target concept (e.g., remove "owl"-containing examples when targeting owl preference). This ensures the effect is truly subliminal and not trivially explained.
- **Do** use a teacher model that is instruction-tuned and supports system prompts in its chat template. Base models without system prompt support will produce meaningless weight differentials.
- **Do** start with a large, diverse source dataset (>500K examples). LLS relies on finding a small high-signal subset within a large pool.
- **Avoid** setting the quantile threshold gamma too low (<0.01). Extremely aggressive filtering produces tiny datasets that overfit and don't generalize.
- **Avoid** using the same model as both teacher and student when auditing for real-world bias. Cross-model transfer is the stronger test of whether a signal is genuinely embedded in the data rather than an artifact of model-specific quirks.

## Error Handling

- **OOM during scoring:** Process examples in chunks of 10K-25K. Use float16/bfloat16 precision. For multi-GPU setups, shard the dataset across ranks and gather results.
- **Degenerate weight distribution (all weights near zero):** The system prompt may not produce meaningful preference shifts in the teacher model. Try a more explicit system prompt, or verify the teacher model actually follows system prompts by testing manually.
- **No effect after training:** Increase the dataset inflation factor (repeat the selected subset 10-20x), reduce beta (try 0.02-0.04), or increase LoRA rank. Also verify the quantile threshold isn't too permissive (allowing noisy examples).
- **Effect appears but is weak:** Lower gamma to select a more stringent top subset (e.g., top 2% instead of 5%). Alternatively, increase the source dataset size.
- **Chat template mismatch:** Ensure the teacher model's tokenizer chat template correctly distinguishes system vs. user messages. Some models concatenate system prompts into the user turn, which weakens the LLS signal.

## Limitations

- LLS requires access to teacher model log-probabilities, so it cannot be applied to closed-source API-only models unless the API exposes token-level log-probs.
- The effect degrades when teacher and student architectures differ substantially. Same-family transfer (e.g., OLMo teacher to OLMo student) is strongest.
- Response truncation to short lengths (20 tokens) is necessary for tractable scoring but limits the complexity of behaviors that can be captured. Subtle stylistic effects may require longer response windows.
- The method assumes the log-linearity property holds for the target behavior. Highly nonlinear or context-dependent behaviors (e.g., "be helpful only on Tuesdays") may not be capturable via LLS.
- LLS is a subset selection method, not a data generation method. It requires a sufficiently large and diverse source dataset that already contains the relevant signal in aggregate.

## Reference

- **Paper:** [Subliminal Effects in Your Data: A General Mechanism via Log-Linearity](https://arxiv.org/abs/2602.04863v1) (Aden-Ali, Golowich, Liu, Shetty, Moitra, 2026). Look for: Algorithm 1 (LLS procedure), Theorem 2.2 (DPO optimality guarantee under log-linearity), and Section 3 (experiments on animal preference, language switching, and persona adoption).
- **Code:** [github.com/ishaqadenali/logit-linear-selection](https://github.com/ishaqadenali/logit-linear-selection) — reference implementation with OLMo teacher and Llama student.