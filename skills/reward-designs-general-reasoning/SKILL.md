---
name: "reward-designs-general-reasoning"
description: "Design and implement likelihood-based reward functions for RL fine-tuning of LLMs on reasoning tasks. Use when: 'design a reward function for RL training', 'implement log-probability rewards for GRPO', 'fine-tune an LLM with likelihood-based rewards', 'build a reward model without a verifier', 'set up RLHF rewards for chain-of-thought', 'implement VeriFree or RLPR-style training'."
---

# Likelihood-Based Reward Designs for General LLM Reasoning

This skill enables Claude to design, implement, and debug reward functions for reinforcement learning fine-tuning of language models on reasoning tasks -- specifically using **log-probability of a reference answer** as the reward signal instead of binary correctness verifiers. The core insight from Kwiatkowski et al. (2026) is that `R(z,a) = log pi_theta(a* | p, z)` -- the log-likelihood of the reference answer given the prompt and chain-of-thought -- is the only reward formulation that works reliably across both verifiable domains (math) and non-verifiable domains (long-form generation), while maintaining alignment with pretraining's next-token prediction objective.

## When to Use

- When the user is implementing RL fine-tuning (GRPO, RLOO, PPO) and needs to choose or implement a reward function for chain-of-thought reasoning
- When building a training pipeline that must work on tasks **without** a binary verifier (proofs, essays, long-form Q&A)
- When the user encounters vanishing reward signals during RL training on long-form generation tasks
- When comparing reward design options (binary vs probability vs log-probability) for a reasoning fine-tuning project
- When implementing VeriFree, JEPO, RLPR, or NOVER and needing guidance on which variant to use
- When the user's RL training shows CoT collapse (chains shrinking to near-zero length) and needs to diagnose the cause
- When setting up reward computation in a GRPO/RLOO training loop with reference answers

## Key Technique

**The problem:** Standard RL fine-tuning of LLMs on reasoning tasks uses binary rewards (1 if correct, 0 otherwise). This requires a task-specific verifier for each benchmark and produces sparse gradient signals. Several recent methods (VeriFree, JEPO, RLPR, NOVER) propose using the model's own likelihood of a reference answer as the reward instead, removing the need for external verifiers. But which likelihood formulation actually works?

**The finding:** The paper systematically tests five reward variants and finds that only **log-probability** rewards work universally. The key distinction is between probability `P(a* | p, z) = prod_t p(a*_t | p, z, a*_{<t})` and log-probability `log P(a* | p, z) = sum_t log p(a*_t | p, z, a*_{<t})`. For long answers, the raw probability vanishes toward zero (it's a product of many values < 1), killing the gradient signal entirely. Log-probability converts the product to a sum, preserving numerical stability and gradient magnitude regardless of answer length. This is also mathematically consistent with the cross-entropy loss used in pretraining, creating a smooth optimization landscape from pretraining through RL fine-tuning.

**Practical impact:** In verifiable settings (math benchmarks), log-probability rewards match or exceed binary reward accuracy while achieving dramatically better perplexity (2.21 vs 67.29 on MATH with Llama-3.2-3B). In non-verifiable settings (long-form answers, proofs), log-probability rewards match SFT performance, while probability-based methods (VeriFree) completely fail. This makes log-probability the single best default reward for any CoT fine-tuning pipeline.

## Step-by-Step Workflow

1. **Define the reward computation function.** Implement `R(z, a) = (1 / |a*|) * sum_t log pi_theta(a*_t | p, z, a*_{<t})` where `p` is the prompt, `z` is the generated chain-of-thought, and `a*` is the reference answer. The `1/|a*|` length normalization prevents bias toward short answers.

2. **Structure the generation format.** Configure the model to output in `<think>...</think><answer>...</answer>` format (DeepSeek-R1 style). The reward is computed only over tokens within `<answer>` tags. The CoT in `<think>` tags is the RL-trained policy output; the answer section is scored against the reference.

3. **Implement the forward pass for reward scoring.** After generating a CoT `z`, concatenate `[prompt, z, reference_answer]` and run a forward pass through the current policy model. Extract the per-token log-probabilities for only the reference answer tokens. Sum them and divide by answer length.

4. **Set up the RL training loop with RLOO advantage estimation.** For each prompt, sample `G` chain-of-thought completions (G=4 minimum, G=32 for best results). Compute the reward for each. The advantage for sample `i` is `A_i = R_i - (1/(G-1)) * sum_{j != i} R_j` (leave-one-out baseline). This is an unbiased estimator that avoids the bias of GRPO's group mean baseline.

5. **Configure the optimizer and schedule.** Use AdamW with learning rate `1e-5`, cosine schedule with 20-step warmup, global gradient norm clipping at 1.0. Batch size should be `num_prompts * G` (e.g., 8 prompts x 32 CoTs = 256 forward passes per step).

6. **Add KL regularization only for verifiable domains.** Apply a KL penalty with coefficient `0.001` against the initial policy when training on tasks with known correct answers (math, code). Omit KL regularization for non-verifiable domains -- it constrains the policy without benefit when the reward is already likelihood-based.

7. **Monitor for CoT collapse.** Track average CoT length during training. In non-verifiable settings, expect CoTs to shrink significantly (to ~10 tokens) -- this is normal behavior where the model learns to skip reasoning and directly emit the answer, effectively converging to SFT. In verifiable settings, CoTs should stabilize at 300-450 tokens.

8. **Evaluate with both greedy accuracy and perplexity.** Use greedy decoding (temperature=0) for accuracy measurement. Compute perplexity as `exp(-(1/|a*|) * log pi^CoT(a* | p))` using Monte Carlo estimation with 32 samples. Log-probability rewards should achieve both competitive accuracy and dramatically better perplexity than binary rewards.

9. **Handle the stop-gradient correctly in the loss.** The full loss is `L = E_z[(R(z) - baseline)^{sg} * log pi(z) + R(z)]` where `^{sg}` is stop-gradient. The first term is the standard REINFORCE policy gradient; the second term is the direct reward optimization. Both are needed when the reward itself depends on model parameters.

10. **Compare against baselines to validate.** Run binary reward and SFT baselines on the same data split. Log-probability rewards should match binary accuracy within 1-2% on verifiable tasks while cutting perplexity by 10-30x. On non-verifiable tasks, they should match SFT perplexity exactly.

## Concrete Examples

**Example 1: Implementing the reward function in PyTorch**

User: "I'm setting up GRPO training and need a reward function that doesn't require a verifier. How do I implement log-probability rewards?"

Approach:
1. Identify that the user needs `R(z, a) = avg log-prob of reference answer tokens`
2. Implement the forward pass that scores reference answers given generated CoTs
3. Integrate into the GRPO/RLOO advantage computation

Output:
```python
import torch
import torch.nn.functional as F

def compute_logprob_reward(model, tokenizer, prompts, cots, reference_answers):
    """
    Compute log-probability reward: R(z,a) = (1/|a*|) * sum log pi(a*_t | p, z, a*_{<t})

    Args:
        model: the policy LLM
        prompts: list of input prompts (str)
        cots: list of generated chain-of-thought strings (str)
        reference_answers: list of ground-truth answer strings (str)
    Returns:
        rewards: tensor of shape (batch,) with per-sample rewards
    """
    rewards = []
    for prompt, cot, ref_answer in zip(prompts, cots, reference_answers):
        # Build the full sequence: prompt + CoT + reference answer
        full_text = prompt + cot + ref_answer
        prefix_text = prompt + cot

        full_ids = tokenizer.encode(full_text, return_tensors="pt").to(model.device)
        prefix_len = len(tokenizer.encode(prefix_text))
        answer_len = full_ids.shape[1] - prefix_len

        if answer_len <= 0:
            rewards.append(torch.tensor(float('-inf')))
            continue

        with torch.no_grad():
            logits = model(full_ids).logits  # (1, seq_len, vocab)

        # Shift: logits[t] predicts token[t+1]
        # We want log P(answer_token_t | everything before it)
        answer_logits = logits[0, prefix_len - 1 : full_ids.shape[1] - 1]  # (answer_len, vocab)
        answer_targets = full_ids[0, prefix_len:]  # (answer_len,)

        log_probs = F.log_softmax(answer_logits, dim=-1)
        token_log_probs = log_probs.gather(1, answer_targets.unsqueeze(1)).squeeze(1)

        # Average log-probability (length-normalized)
        avg_logprob = token_log_probs.sum() / answer_len
        rewards.append(avg_logprob)

    return torch.stack(rewards)


def rloo_advantages(rewards_per_prompt, G):
    """
    RLOO leave-one-out advantage estimation.

    Args:
        rewards_per_prompt: tensor of shape (G,) -- rewards for G CoTs on one prompt
        G: number of samples per prompt
    Returns:
        advantages: tensor of shape (G,)
    """
    total = rewards_per_prompt.sum()
    # Leave-one-out baseline for sample i = (total - R_i) / (G - 1)
    baselines = (total - rewards_per_prompt) / (G - 1)
    return rewards_per_prompt - baselines
```

**Example 2: Diagnosing training failure with probability-based rewards**

User: "My VeriFree training works fine on MATH but completely flatlines on my long-form proof generation task. The reward barely changes from epoch 1."

Approach:
1. Identify the root cause: probability rewards vanish for long sequences
2. Compute the magnitude of raw probabilities to confirm
3. Recommend switching to log-probability rewards

Output:
```
The issue is that VeriFree uses R = pi(a* | p, z), the raw probability of the
reference answer. For a proof that is 200 tokens long, even with 95% per-token
accuracy, the total probability is 0.95^200 ≈ 3.5e-5. For realistic per-token
accuracies (~60-80%), this is effectively zero.

Diagnosis: Add logging to check the reward magnitude:

    print(f"Raw prob reward: {reward.item():.2e}")
    # You will likely see values < 1e-10 for long proofs

Fix: Switch from probability to log-probability rewards:

    # BEFORE (VeriFree-style, fails on long sequences):
    reward = torch.exp(token_log_probs.sum())  # product of probs -> vanishes

    # AFTER (log-probability, works universally):
    reward = token_log_probs.sum() / answer_length  # sum of log-probs -> stable

This single change converts a multiplicative (vanishing) signal to an additive
(stable) signal. The gradient magnitude scales linearly with answer length
instead of exponentially decaying.
```

**Example 3: Choosing the right reward for a mixed-domain training run**

User: "I'm training on a mix of math problems (with known answers) and open-ended coding explanations (no verifier). What reward should I use?"

Approach:
1. Identify this as a mixed verifiable/non-verifiable setting
2. Recommend log-probability as the only reward that handles both
3. Provide configuration guidance

Output:
```
Use log-probability rewards for both domains. This is the only reward type from
the literature that works reliably across verifiable and non-verifiable tasks.

Configuration for your mixed-domain setup:

    reward_config = {
        "reward_type": "avg_logprob",           # (1/|a*|) * sum log pi(a*_t | ...)
        "kl_coefficient": 0.001,                 # Apply KL only to math subset
        "kl_domains": ["math"],                  # No KL for open-ended domains
        "group_size": 32,                        # Samples per prompt for RLOO
        "max_cot_length": 1024,                  # Token budget for reasoning
        "answer_format": "<answer>{answer}</answer>",
    }

Key settings:
- Use KL regularization (0.001) ONLY for the math domain where you have
  verifiable answers. Omit it for the coding explanations.
- Expect different CoT behavior per domain: math CoTs will be 300-400 tokens,
  explanation CoTs will collapse to ~10 tokens (the model learns to skip
  reasoning and directly produce the answer -- this is expected and matches
  SFT performance).
- Do NOT mix binary rewards for math + logprob for explanations. Using a single
  reward type across domains simplifies training dynamics and avoids reward
  scale mismatches.
```

## Best Practices

- **Do:** Always use **log**-probability, never raw probability. The `log` is not optional -- it prevents vanishing rewards on any sequence longer than ~20 tokens.
- **Do:** Length-normalize the reward by dividing `sum log pi(a*_t)` by `|a*|`. This prevents systematic bias toward short answers and makes rewards comparable across samples with different answer lengths.
- **Do:** Use RLOO (leave-one-out) advantage estimation instead of GRPO's group mean. RLOO is unbiased while GRPO's baseline introduces bias that can interact poorly with likelihood-based rewards.
- **Do:** Use greedy decoding (temperature=0) at evaluation time. Log-probability-trained models show a gap between greedy and sampled accuracy -- greedy is consistently better.
- **Avoid:** Adding KL regularization in non-verifiable domains. The log-probability reward already acts as an implicit regularizer toward the data distribution; additional KL over-constrains the policy.
- **Avoid:** Using a reward penalty to artificially maintain CoT length. Length penalties stabilize CoT length but degrade task performance. CoT collapse in non-verifiable settings is the correct behavior -- the model is learning that reasoning doesn't help for these tasks.

## Error Handling

**Reward is NaN or -inf:** This occurs when the reference answer contains tokens with zero probability under the model. Add a small epsilon floor: `log_prob = max(log_prob, -100.0)` per token, or filter training samples where the initial model assigns negligible probability to the reference.

**CoT collapses to empty string:** If CoTs shrink to 0 tokens (not just short), check that the `<think>` tag is being generated correctly and that the stop condition is not triggering prematurely on the `<answer>` tag. Ensure the model sees the full `<think>...</think><answer>` template during training.

**Reward variance is too high across the group:** With G=4, RLOO advantages may be noisy. Increase to G=16 or G=32 for stable training, at the cost of more compute per step. Gradient clipping at 1.0 also helps absorb reward variance.

**Training diverges after initial progress:** This typically indicates the learning rate is too high for the reward scale. Log-probability rewards have larger magnitude than binary [0,1] rewards. Reduce learning rate to `5e-6` or add gradient norm clipping if not already present.

## Limitations

- **CoT quality in non-verifiable settings:** Log-probability rewards match SFT accuracy on non-verifiable tasks but do not exceed it. The CoT collapses, meaning the model does not learn to reason -- it learns to directly produce the answer. If you need genuine chain-of-thought reasoning on open-ended tasks, this method alone is insufficient.
- **Requires reference answers:** Unlike binary verifiers that can check arbitrary solutions, log-probability rewards require a specific reference answer string. Multiple valid answers (e.g., different correct proofs) require either multiple reference samples or a canonical form.
- **Reward depends on the policy model:** Because the reward is `log pi_theta(a*)`, it changes as the model updates. This introduces non-stationarity that RLOO handles but simpler baselines (fixed reward model) avoid. The stop-gradient formulation in the loss is critical to stability.
- **Temperature sensitivity:** Models trained with log-probability rewards show better greedy performance but may underperform at temperature=1 sampling compared to binary-reward models. Use greedy or low-temperature decoding in production.
- **Scale tested:** The paper validates on 3B parameter models. Behavior at larger scales (70B+) is extrapolated but not confirmed.

## Reference

**Paper:** Kwiatkowski, A., Butt, N., Labiad, I., Kempe, J., & Ollivier, Y. (2026). *Likelihood-Based Reward Designs for General LLM Reasoning.* arXiv:2602.03979v1. [https://arxiv.org/abs/2602.03979v1](https://arxiv.org/abs/2602.03979v1)

**Key takeaway:** Table 1 and Table 2 contain the comprehensive comparison of all reward variants across verifiable and non-verifiable domains. Figure 3 shows why probability rewards fail (vanishing signal visualization). Section 4.2 provides the critical analysis of CoT collapse behavior.