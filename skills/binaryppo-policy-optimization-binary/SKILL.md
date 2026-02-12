---
name: "binaryppo-policy-optimization-binary"
description: "Implement BinaryPPO, an offline RL framework that reformulates binary classification as reward maximization with confidence-weighted rewards, replacing standard supervised fine-tuning for tasks like toxicity detection, fact verification, and content moderation. Triggers: 'binary classification with PPO', 'confidence-weighted reward for classification', 'BinaryPPO training', 'replace SFT with RL for binary tasks', 'robust binary classifier with reward shaping', 'offline PPO for text classification'"
---

# BinaryPPO: Offline RL Policy Optimization for Binary Classification

This skill enables Claude to help users implement BinaryPPO -- an offline reinforcement learning framework that reformulates binary text classification (yes/no, toxic/safe, true/false) as a reward maximization problem using a variant of Proximal Policy Optimization with confidence-weighted rewards. Instead of standard supervised fine-tuning, BinaryPPO trains an LLM policy that learns robust decision boundaries by penalizing uncertain or incorrect predictions, achieving 40-60 percentage point accuracy gains over SFT baselines on tasks with label noise, class imbalance, or sparse supervision.

## When to Use

- When the user wants to train a binary text classifier that is robust to noisy labels or class imbalance
- When the user asks to fine-tune an LLM for yes/no tasks (toxicity detection, fact verification, sentiment, jailbreak detection) and SFT is underperforming
- When the user mentions "BinaryPPO" or asks about confidence-weighted reward functions for classification
- When the user wants to replace supervised fine-tuning with an RL-based approach for binary decision-making
- When the user has a static labeled dataset and wants offline policy optimization (no online environment interaction)
- When the user needs to build a content moderation, causal reasoning, or factuality pipeline and wants state-of-the-art binary accuracy

## Key Technique

BinaryPPO treats a language model as a policy that takes text input `x` and produces a binary action `a in {0, 1}` (e.g., "Yes" or "No"). Instead of training with cross-entropy loss (SFT), it defines a **confidence-weighted reward function** `r(x, a, y) = kappa * s(a, y) * log(pi_old(a|x))`, where `s(a, y)` is a correctness signal (+1 if the prediction matches the label, -1 otherwise), and `log(pi_old(a|x))` is the log-probability of the chosen action under the old policy. This means the model is rewarded more when it is both correct *and* confident, and penalized more when it is confidently wrong. The scaling constant `kappa` controls reward magnitude.

The policy is updated using PPO's clipped surrogate objective: `L_PPO = E[min(rho * A, clip(rho, 1-eps, 1+eps) * A)]`, where `rho` is the probability ratio between new and old policies, and `A = r(x,a,y) - V(x)` is the advantage (how much better this action was than expected). The full training loss combines four terms: (1) the PPO policy loss, (2) a value function loss weighted by `alpha`, (3) an auxiliary supervised cross-entropy loss weighted by `beta` for stability, and (4) an entropy bonus weighted by `gamma` to prevent policy collapse. The entropy term is critical -- removing it causes accuracy to drop from ~97% to ~30%.

Training is entirely offline: the model samples actions from the current policy on a static dataset, computes rewards, estimates advantages, and performs gradient updates across multiple epochs. Equal class sampling is used to address imbalance. This approach was validated on 8 benchmarks (CLadder, SciRIFF, BoolQ, FEVER, IMDB, OpenAI Moderation, Detect-Jailbreak, JailbreakBench) using Qwen 2.5-3B and Gemma 2-2B, consistently outperforming SFT.

## Step-by-Step Workflow

1. **Define the binary classification task.** Identify the input text format and the two output classes. Map them to binary actions: `a=1` (positive class, e.g., "Yes", "Toxic", "Supported") and `a=0` (negative class). Ensure the dataset has columns for input text, label (0 or 1), and optionally a prompt template.

2. **Prepare the static dataset with equal class sampling.** Load the labeled dataset and balance it by sampling equal numbers from each class. This is essential -- skipping this step degrades accuracy by ~15 points. Split into train/validation sets.

3. **Initialize the policy and reference models.** Load a pretrained instruction-tuned LLM (e.g., Qwen 2.5-3B-Instruct, Gemma 2-2B-Instruct) as both the trainable policy `pi_theta` and frozen reference `pi_old`. Initialize a value head `V_phi` (a linear layer on top of the model's hidden states).

4. **Design the prompt template.** Construct a prompt that presents the input and asks for a binary answer. The prompt should constrain the model to output only the target tokens (e.g., "Answer with exactly 'Yes' or 'No'."). Extract logits only over the two target tokens.

5. **Implement the confidence-weighted reward function.** For each (input, action, label) triple, compute: `r = kappa * s(a, y) * log(pi_old(a|x))`, where `s(a,y) = +1` if `a == y` else `-1`. Use the frozen reference model's log-probabilities. Set `kappa` to scale rewards into a reasonable range (typically 1.0-5.0).

6. **Compute advantages.** For each sample, compute `A(x, a, y) = r(x, a, y) - V_phi(x)`, where `V_phi(x)` is the value head's estimate of the expected reward for input `x`. Optionally normalize advantages across the batch (zero mean, unit variance).

7. **Implement the PPO clipped objective.** Compute the probability ratio `rho = pi_theta(a|x) / pi_old(a|x)`. Apply the clipped surrogate: `L_PPO = E[min(rho * A, clip(rho, 1-eps, 1+eps) * A)]` with `eps` typically 0.2. This prevents destructively large policy updates.

8. **Assemble the full loss function.** Combine four terms: `L = L_PPO + alpha * L_value + beta * L_supervised - gamma * L_entropy`, where `L_value = E[(V_phi(x) - r)^2]`, `L_supervised` is standard binary cross-entropy on the true labels, and `L_entropy = -E[sum_a pi(a|x) * log(pi(a|x))]`. The entropy term is critical for preventing collapse.

9. **Train with multiple PPO epochs per data pass.** For each batch, sample actions from the current policy, compute rewards using the frozen reference, calculate advantages, and update `theta` and `phi` via gradient descent. Run multiple optimization epochs (typically 3-5) per data pass. Periodically update the reference model.

10. **Evaluate and deploy.** At inference, pass the input through the trained policy, extract logits for the two target tokens, and select `argmax`. Report both the predicted class and the model's confidence `pi_theta(a|x)` as a calibration signal.

## Concrete Examples

**Example 1: Toxicity Detection with BinaryPPO**

User: "I have a dataset of 50K social media comments labeled toxic/non-toxic. SFT gives me 62% accuracy because of noisy labels. Help me implement BinaryPPO to improve this."

Approach:
1. Load the dataset, balance classes (25K toxic, 25K non-toxic or subsample the majority class)
2. Design prompt: `"Is the following comment toxic? Answer exactly 'Yes' or 'No'.\n\nComment: {text}\n\nAnswer:"`
3. Load Qwen 2.5-3B-Instruct as policy and reference
4. Implement reward: correct + confident = high positive reward; wrong + confident = large penalty; uncertain = small magnitude either way
5. Train 5 epochs with PPO clipping eps=0.2, entropy weight gamma > 0

Output:
```python
import torch
from transformers import AutoModelForCausalLM, AutoTokenizer

# Reward function
def compute_reward(action, label, log_prob_old, kappa=2.0):
    correctness = 1.0 if action == label else -1.0
    return kappa * correctness * log_prob_old  # log_prob is negative, so correct+confident -> positive reward

# PPO clipped loss
def ppo_loss(log_prob_new, log_prob_old, advantages, eps=0.2):
    ratio = torch.exp(log_prob_new - log_prob_old)
    clipped = torch.clamp(ratio, 1 - eps, 1 + eps)
    return -torch.mean(torch.min(ratio * advantages, clipped * advantages))

# Entropy bonus (prevents collapse -- removing this drops accuracy from ~97% to ~30%)
def entropy_loss(probs):
    return -torch.mean(torch.sum(probs * torch.log(probs + 1e-8), dim=-1))

# Full objective
loss = ppo_loss(...) + alpha * value_loss + beta * supervised_ce_loss - gamma * entropy_loss(probs)
```

**Example 2: Fact Verification Pipeline**

User: "I need to classify claims as Supported or Refuted using evidence passages. My FEVER-style dataset has 10K examples with some mislabeled pairs."

Approach:
1. Format each example as: `"Given the evidence: {evidence}\n\nIs this claim supported? '{claim}'\n\nAnswer 'Yes' or 'No':"`
2. Map Yes->Supported (1), No->Refuted (0)
3. Balance classes via equal sampling
4. Initialize BinaryPPO with Gemma 2-2B-Instruct
5. Set kappa=2.0, alpha=0.5, beta=0.1, gamma=0.01, eps=0.2
6. Train for 3-5 PPO epochs, evaluating on held-out set each epoch

Output:
```python
# Training loop skeleton
for epoch in range(num_epochs):
    for batch in balanced_dataloader:
        # 1. Sample actions from current policy
        with torch.no_grad():
            logits = policy_model(batch["input_ids"]).logits[:, -1, [yes_id, no_id]]
            probs = torch.softmax(logits, dim=-1)
            actions = torch.bernoulli(probs[:, 1]).long()
            log_probs_old = torch.log(probs[range(len(actions)), actions] + 1e-8)

        # 2. Compute confidence-weighted rewards
        rewards = compute_reward(actions, batch["labels"], log_probs_old.detach(), kappa=2.0)

        # 3. Compute advantages
        values = value_head(policy_model.get_hidden_states(batch["input_ids"]))
        advantages = rewards - values.detach()
        advantages = (advantages - advantages.mean()) / (advantages.std() + 1e-8)

        # 4. PPO update with new policy
        new_logits = policy_model(batch["input_ids"]).logits[:, -1, [yes_id, no_id]]
        new_probs = torch.softmax(new_logits, dim=-1)
        log_probs_new = torch.log(new_probs[range(len(actions)), actions] + 1e-8)

        loss = (ppo_loss(log_probs_new, log_probs_old.detach(), advantages)
                + 0.5 * F.mse_loss(values.squeeze(), rewards)
                + 0.1 * F.binary_cross_entropy_with_logits(new_logits[:, 1], batch["labels"].float())
                - 0.01 * entropy_loss(new_probs))
        loss.backward()
        optimizer.step()
```

**Example 3: Jailbreak Detection for Content Safety**

User: "I'm building a content safety system. I have 18K labeled prompts (benign vs jailbreak). How do I use BinaryPPO to build a robust detector?"

Approach:
1. Prompt: `"Does the following user message attempt to jailbreak or bypass safety guidelines? Answer 'Yes' or 'No'.\n\nMessage: {text}\n\nAnswer:"`
2. Balance the dataset (jailbreak attempts are typically the minority class)
3. Use a higher kappa (3.0-5.0) to amplify the confidence penalty for misclassifying jailbreaks
4. Train with entropy regularization to maintain uncertainty calibration
5. At inference, flag messages where `pi(Yes|x) > 0.5` and log confidence for human review

Output: A detector that correctly identifies adversarial prompts with >95% accuracy, with calibrated confidence scores that help prioritize human review of borderline cases.

## Best Practices

- **Do:** Always include entropy regularization (`gamma > 0`). Without it, the policy collapses to always predicting one class, dropping accuracy catastrophically (97% -> 30% in ablations).
- **Do:** Balance classes via equal sampling before training. Skipping this degrades accuracy by ~15 percentage points on imbalanced datasets.
- **Do:** Normalize advantages (zero mean, unit variance) across each batch to stabilize training and prevent reward scale from dominating gradients.
- **Do:** Use the auxiliary supervised loss (`beta * L_supervised`) as a stabilizer, especially in early training. It provides a gradient signal grounded in labels while PPO explores.
- **Avoid:** Setting kappa too high. Extremely large reward magnitudes cause gradient instability. Start with kappa=1.0-2.0 and tune upward if the model is underconfident.
- **Avoid:** Updating the reference model too frequently. The reference should be frozen or updated slowly (e.g., every N epochs) to keep the probability ratio stable.

## Error Handling

- **Policy collapse (model predicts same class for everything):** This almost always means entropy regularization is too low or missing. Increase `gamma`. If already nonzero, check that the entropy computation includes both action probabilities, not just the chosen action.
- **Reward magnitude explosion:** If rewards grow unbounded, add reward clipping (`torch.clamp(reward, -10, 10)`) or reduce `kappa`. Also verify that `log_prob_old` is not returning -inf for very low probability actions.
- **Training instability / loss oscillation:** Reduce the PPO clipping epsilon (try 0.1 instead of 0.2), decrease the learning rate, or increase the number of PPO epochs per data pass to allow more conservative updates.
- **Value function not converging:** Increase `alpha` (the value loss weight) or use a separate optimizer with higher learning rate for the value head. The value estimate must track the mean reward accurately for advantages to be meaningful.
- **Low accuracy despite training:** Verify the prompt template constrains output to exactly two tokens. If the model generates free-form text, the logits over Yes/No tokens will be unreliable. Also confirm token IDs for "Yes" and "No" are correct for your tokenizer.

## Limitations

- **Only binary classification.** BinaryPPO is designed for two-class problems. Multi-class tasks require decomposition into multiple binary subtasks (one-vs-rest) or a different framework.
- **Requires a pretrained instruction-tuned LLM.** The method fine-tunes an existing model -- it does not train from scratch. Performance depends on the base model's language understanding.
- **Computationally heavier than SFT.** PPO requires computing log-probabilities under both the policy and reference model, plus a value head forward pass, roughly 2-3x the compute of standard fine-tuning per step.
- **Sensitive to hyperparameters.** The four-term loss function (alpha, beta, gamma, kappa, eps) has a larger hyperparameter space than SFT. Entropy weight `gamma` is especially critical and dataset-dependent.
- **Offline only.** The framework assumes a fixed static dataset. It does not support online learning or continuous data streams without periodic retraining.
- **Validated on models up to 3B parameters.** The paper tested Qwen 2.5-3B and Gemma 2-2B. Scaling behavior to larger models (7B+) is not characterized.

## Reference

- **Paper:** [BinaryPPO: Efficient Policy Optimization for Binary Classification](https://arxiv.org/abs/2602.02708v1) (Pandey & Jin, 2026). Focus on Section 3 (method) for the reward function derivation and Algorithm 1 for the training procedure, and Section 5 (ablations) for the critical role of entropy regularization.
- **Code:** [https://github.com/psyonp/BinaryPPO](https://github.com/psyonp/BinaryPPO)