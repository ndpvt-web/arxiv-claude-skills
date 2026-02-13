---
name: "rewards-as-labels-revisiting"
description: "Implement the REAL (Rewards as Labels) framework for LLM reinforcement learning, which reformulates RLVR policy optimization as classification instead of scalar-weighted policy gradient. Use when: 'implement REAL loss for RLVR', 'fix gradient issues in GRPO training', 'classification-based reward optimization', 'replace GRPO with bounded gradient method', 'train reasoning model with verifiable rewards', 'implement anchor logit loss function'."
---

# Rewards as Labels (REAL): Classification-Based RLVR for LLM Training

This skill enables Claude to implement the REAL framework from Zhai et al. (2026), which recasts Reinforcement Learning with Verifiable Rewards (RLVR) as a classification problem. Instead of treating reward scores as scalar advantage weights in policy gradients (as GRPO/DAPO do), REAL converts binary verifiable rewards into categorical labels and optimizes a cross-entropy objective over length-normalized relative log-probabilities. This produces monotonically decreasing, bounded gradient weights that fix two critical pathologies in standard RLVR: gradient misassignment in correct rollouts and gradient domination by incorrect rollouts.

## When to Use

- When implementing or improving RLVR training loops for reasoning LLMs (math, code verification, formal proofs)
- When GRPO or DAPO training is unstable, showing loss spikes or reward hacking from unbounded gradients
- When building a custom training pipeline that uses binary verifiable rewards (correct/incorrect)
- When the user wants to replace advantage-weighted policy gradient losses with a classification-based alternative
- When debugging gradient magnitude issues in multi-rollout RL training where some samples dominate updates
- When implementing anchor logit mechanisms to enforce separation between positive and negative rollout scores

## Key Technique

**The Problem with GRPO-style Methods.** Standard RLVR methods like GRPO compute per-token gradient weights proportional to `|A * exp(s_t)|`, where `A` is the advantage and `s_t` is the token-level log-ratio. This creates two pathologies: (1) *Gradient Misassignment in Positives* -- tokens that already have high probability under the policy receive disproportionately large updates because `exp(s_t)` is large, while low-probability tokens that most need learning get weak gradients; (2) *Gradient Domination in Negatives* -- a few negative rollouts with high relative log-probability produce unbounded gradient magnitudes that swamp the entire batch update.

**REAL's Classification Reformulation.** REAL partitions each batch of G rollouts for a prompt into positive set `O+` (reward=1) and negative set `O-` (reward=0) based on a verifiable checker. It computes a length-normalized relative log-probability for each rollout: `s_bar_k = (1/|o_k|) * sum_t log(pi_theta(o_t|q) / pi_old(o_t|q))`. Then it optimizes a softmax cross-entropy loss with a fixed anchor logit at 0:

```
L_REAL = log(1 + sum_{O+} exp(-s_bar_i / tau)) + log(1 + sum_{O-} exp(s_bar_j / tau))
```

The anchor at 0 acts as a negative reference for positive rollouts and a positive reference for negative rollouts, enforcing that correct solutions score above 0 and incorrect ones score below 0. The resulting gradient weight for any rollout is bounded by `1/tau` and monotonically decreases as the rollout's relative probability increases -- meaning the model focuses learning on the rollouts it finds hardest, not the ones it already gets right.

**Why It Works.** The bounded `1/tau` ceiling prevents any single rollout from dominating the gradient. The monotonic decrease ensures low-probability correct rollouts (the ones the model most needs to learn) receive the strongest positive signal, while high-probability incorrect rollouts (the most dangerous errors) receive the strongest suppression. No explicit KL penalty is needed because the loss structure inherently regularizes.

## Step-by-Step Workflow

1. **Set up the verifiable reward function.** Implement a binary checker that returns 1 (correct) or 0 (incorrect) for each generated rollout. For math tasks, this is typically answer extraction + exact match. For code, it is test-suite execution.

2. **Generate G rollouts per prompt.** Sample G completions (paper uses G=8) from the current policy `pi_theta` for each prompt in the batch. Store the per-token log-probabilities under both `pi_theta` and the reference policy `pi_old`.

3. **Partition rollouts into positive and negative sets.** Apply the verifier to each rollout. Assign rollouts with reward=1 to `O+` and reward=0 to `O-`. Skip prompts where all rollouts are the same label (all correct or all incorrect) as they provide no contrastive signal.

4. **Compute length-normalized relative log-probability (logits).** For each rollout k, calculate:
   ```python
   s_bar_k = (1 / len(output_k)) * sum(
       log_pi_theta[t] - log_pi_old[t] for t in range(len(output_k))
   )
   ```

5. **Compute the REAL loss with anchor logits.** Use temperature `tau=0.5`:
   ```python
   loss_pos = torch.log(1 + sum(torch.exp(-s_bar_i / tau) for i in O_plus))
   loss_neg = torch.log(1 + sum(torch.exp(s_bar_j / tau) for j in O_minus))
   loss = loss_pos + loss_neg
   ```
   The `1 +` terms represent the anchor logit fixed at 0.

6. **Backpropagate and update.** The gradient with respect to each rollout's tokens is automatically bounded by `1/tau` and weighted inversely to how well the model already handles that rollout. Use learning rate `2e-6` for 1.5B models, `1e-6` for 7B models.

7. **Iterate without KL penalty.** Unlike GRPO, no explicit KL divergence term is needed. The anchor logit and bounded gradients provide implicit regularization. Update `pi_old` periodically (e.g., every training step or every few steps).

8. **Monitor training stability.** Track the mean logit scores for `O+` and `O-` separately. Healthy training shows positive logits increasing and negative logits decreasing, with both magnitudes staying moderate (not exploding).

## Concrete Examples

**Example 1: Implementing the REAL loss function in PyTorch**

User: "I'm training a 1.5B reasoning model with GRPO but getting unstable gradients. Can you implement the REAL loss as a replacement?"

Approach:
1. Replace the advantage-weighted clipped policy gradient with classification loss
2. Implement length-normalized logit computation
3. Add anchor logit via the log(1 + ...) formulation

Output:
```python
import torch
import torch.nn.functional as F

def real_loss(
    log_probs_theta: torch.Tensor,   # (G, max_seq_len) current policy log-probs
    log_probs_old: torch.Tensor,      # (G, max_seq_len) reference policy log-probs
    rewards: torch.Tensor,            # (G,) binary rewards from verifier
    seq_lengths: torch.Tensor,        # (G,) actual sequence lengths
    tau: float = 0.5,
) -> torch.Tensor:
    """REAL: Rewards as Labels classification loss for RLVR."""
    # Step 1: Compute per-token log ratios
    log_ratios = log_probs_theta - log_probs_old  # (G, max_seq_len)

    # Step 2: Mask padding and compute length-normalized logits
    mask = torch.arange(log_ratios.size(1), device=log_ratios.device).unsqueeze(0)
    mask = (mask < seq_lengths.unsqueeze(1)).float()
    s_bar = (log_ratios * mask).sum(dim=1) / seq_lengths.float()  # (G,)

    # Step 3: Partition into positive and negative sets
    pos_mask = rewards == 1
    neg_mask = rewards == 0

    loss = torch.tensor(0.0, device=log_ratios.device)

    # Step 4: Positive loss -- anchor at 0 acts as negative reference
    if pos_mask.any():
        pos_logits = s_bar[pos_mask] / tau
        # log(1 + sum(exp(-s_bar_i / tau))) -- anchor contributes the "1"
        loss = loss + torch.log(1.0 + torch.exp(-pos_logits).sum())

    # Step 5: Negative loss -- anchor at 0 acts as positive reference
    if neg_mask.any():
        neg_logits = s_bar[neg_mask] / tau
        # log(1 + sum(exp(s_bar_j / tau)))
        loss = loss + torch.log(1.0 + torch.exp(neg_logits).sum())

    return loss
```

**Example 2: Integrating REAL into an existing GRPO training loop**

User: "I have a GRPO training loop using trl. How do I swap in REAL?"

Approach:
1. Identify where the GRPO advantage-weighted loss is computed
2. Replace with REAL loss computation
3. Remove KL penalty since REAL doesn't need it

Output:
```python
# BEFORE (GRPO-style):
# advantages = compute_advantages(rewards)  # group-normalized
# ratio = (log_probs - old_log_probs).exp()
# clipped = torch.clamp(ratio, 1-eps, 1+eps)
# loss = -torch.min(ratio * advantages, clipped * advantages).mean()
# loss = loss + beta * kl_divergence  # KL penalty

# AFTER (REAL):
def compute_real_loss_for_batch(batch, tau=0.5):
    """Drop-in replacement for GRPO loss computation."""
    losses = []
    for prompt_idx in range(batch["prompt_ids"].size(0)):
        log_p = batch["log_probs_theta"][prompt_idx]   # (G, seq_len)
        log_old = batch["log_probs_old"][prompt_idx]    # (G, seq_len)
        rewards = batch["rewards"][prompt_idx]           # (G,)
        lengths = batch["seq_lengths"][prompt_idx]       # (G,)

        # Skip if all same reward (no contrastive signal)
        if rewards.all() or (~rewards.bool()).all():
            continue

        losses.append(real_loss(log_p, log_old, rewards, lengths, tau))

    if not losses:
        return torch.tensor(0.0, requires_grad=True)
    return torch.stack(losses).mean()
    # No KL penalty needed -- anchor logits provide implicit regularization
```

**Example 3: Binary cross-entropy variant (simpler alternative)**

User: "I want the simplest version of REAL. Can I just use BCE?"

Approach:
1. Treat each rollout independently with sigmoid instead of softmax over the group
2. Still use length-normalized logits and anchor at 0

Output:
```python
def real_bce_loss(
    log_probs_theta: torch.Tensor,
    log_probs_old: torch.Tensor,
    rewards: torch.Tensor,
    seq_lengths: torch.Tensor,
    tau: float = 0.5,
) -> torch.Tensor:
    """REAL-BCE: Simpler binary cross-entropy variant.
    Slightly weaker than full REAL (~4.5% over DAPO vs ~6.7%)
    but simpler and still stable."""
    log_ratios = log_probs_theta - log_probs_old
    mask = torch.arange(log_ratios.size(1), device=log_ratios.device).unsqueeze(0)
    mask = (mask < seq_lengths.unsqueeze(1)).float()
    s_bar = (log_ratios * mask).sum(dim=1) / seq_lengths.float()

    # BCE: each rollout independently classified against anchor
    logits = s_bar / tau
    loss = F.binary_cross_entropy_with_logits(logits, rewards.float())
    return loss
```

## Best Practices

- **Do:** Use `tau=0.5` as the default temperature. The paper validates this across both 1.5B and 7B scales. Lower values sharpen gradients; higher values flatten them.
- **Do:** Skip prompts where all G rollouts have the same reward. These provide no contrastive signal and add noise. This is consistent with group-relative methods.
- **Do:** Use length normalization when computing logits. Without it, longer sequences produce larger magnitudes that distort the loss landscape.
- **Do:** Use `logsumexp` tricks for numerical stability when implementing the loss with large G or small tau values (e.g., `torch.logsumexp` instead of `log(1 + sum(exp(...)))`).
- **Avoid:** Adding a KL divergence penalty. REAL's anchor logit mechanism provides implicit regularization. Adding explicit KL is redundant and can hurt performance.
- **Avoid:** Using REAL with non-binary rewards without first discretizing them. The method assumes categorical labels. For continuous rewards, threshold into bins first.

## Error Handling

- **All rollouts same label:** When every rollout for a prompt is correct (or all incorrect), the loss degenerates. Skip these prompts or return zero loss. Log the skip rate -- if it exceeds 50%, the model may be too strong or too weak for the task distribution.
- **Numerical overflow in exp:** When `s_bar / tau` is large, `exp(s_bar / tau)` overflows. Use `torch.logsumexp` to compute the loss in log-space:
  ```python
  # Instead of: log(1 + sum(exp(x)))
  # Use: torch.logsumexp(torch.cat([torch.zeros(1), x]), dim=0)
  ```
- **Reference policy drift:** If `pi_old` drifts too far from `pi_theta`, logit magnitudes grow and training destabilizes. Update `pi_old` every step or every few steps.
- **Empty positive or negative set:** When partitioning produces an empty set for one side, compute loss only for the non-empty set. This is valid -- the anchor still provides a reference point.

## Limitations

- **Binary rewards only (natively).** REAL assumes rewards are 0 or 1 from a verifiable checker. Extending to continuous or multi-level rewards requires discretization, which loses information.
- **Requires a verifiable reward function.** Tasks without automated correctness checking (open-ended generation, creative writing, subjective quality) cannot use this method directly.
- **Group size sensitivity.** With very small G (e.g., G=2), the positive/negative partition may frequently be all-one-class, reducing effective training data. G=8 is the validated default.
- **Tested on math reasoning.** The paper validates on mathematical benchmarks (AIME, MATH, AMC, Minerva, Olympiad Bench). Generalization to code generation, formal verification, or multi-step tool use is plausible but unverified.
- **Requires reference policy storage.** Like GRPO, REAL needs `pi_old` log-probabilities, doubling memory for the forward pass compared to pure supervised fine-tuning.

## Reference

**Paper:** Zhai, Z., Chen, M., Zhao, J., Qian, J., & Shen, L. (2026). *Rewards as Labels: Revisiting RLVR from a Classification Perspective.* [arXiv:2602.05630](https://arxiv.org/abs/2602.05630v1)

**Key sections to read:** Section 4 (gradient analysis of GRPO showing the two pathologies), Section 5 (REAL formulation and Proposition 5.1 proving bounded monotonic gradients), Algorithm 1 (full training procedure), and Table 1 (benchmark results across model scales).