---
name: "dispo-enhancing-training-efficiency"
description: "Implement the DISPO reinforcement learning algorithm for training LLMs on mathematical reasoning with decoupled importance sampling clipping. Use when: 'implement DISPO training loop', 'set up RL fine-tuning for math reasoning', 'configure clipping for GRPO/REINFORCE', 'debug training collapse in RL for LLMs', 'tune exploration vs distillation in policy optimization', 'fix repetition or length collapse in RL training'."
---

# DISPO: Decoupled Importance Sampling Policy Optimization for LLM Math Reasoning

This skill enables Claude to implement, configure, and debug the DISPO algorithm -- a REINFORCE-style reinforcement learning method that decouples importance sampling (IS) weight clipping into four independent regimes based on response correctness and weight direction. DISPO achieves both fast learning (like CISPO) and stable training (like PPO/GRPO) by separately controlling exploration amplification, distillation suppression, incorrect-response unlearning, and over-correction prevention. It reached 61.04% on AIME'24 vs. 55.42% for CISPO and 50.21% for DAPO.

## When to Use

- When the user wants to implement an RL training loop for LLM mathematical reasoning with verifiable rewards
- When the user is experiencing training instability (repetition collapse, length collapse, or gradual accuracy degradation) in an existing GRPO/DAPO/CISPO setup
- When the user asks to configure or tune importance sampling clipping parameters for policy optimization
- When the user needs to balance exploration vs. distillation during RL fine-tuning of language models
- When the user wants to convert a PPO-style (GRPO/DAPO) training pipeline to a faster REINFORCE-style approach without sacrificing stability
- When the user asks to diagnose why their RL-trained model produces repetitive outputs or vanishingly short responses

## Key Technique

**The core insight:** Standard RL methods for LLMs apply a single clipping window to importance sampling weights regardless of whether the response was correct or incorrect. DISPO decouples this into four independent parameters by crossing two axes: (1) whether the advantage is positive (correct response) or negative (incorrect response), and (2) whether the IS weight is above 1 (policy moved away from reference) or below 1 (policy moved toward reference). This yields four regimes, each with a distinct and independently tunable effect on training dynamics.

**The four regimes and their effects:**

| Regime | Condition | Parameter | Controls |
|--------|-----------|-----------|----------|
| 1 | Correct response, weight > 1 | `epsilon_plus_high` | Exploration: amplifies entropy by reinforcing unlikely-but-correct tokens |
| 2 | Correct response, weight < 1 | `epsilon_plus_low` | Distillation: reduces entropy by concentrating probability on high-confidence correct tokens |
| 3 | Incorrect response, weight > 1 | `epsilon_minus_high` | Unlearning: penalizes high-probability incorrect tokens the model is confident about |
| 4 | Incorrect response, weight < 1 | `epsilon_minus_low` | Over-correction guard: prevents excessive penalization of already-unlikely incorrect tokens |

**The decoupled clipping function** replaces the standard uniform clip:

```
r_clipped(theta) = {
  clip(r(theta), 1 - eps_plus_low,  1 + eps_plus_high)   if advantage > 0  (correct)
  clip(r(theta), 1 - eps_minus_low, 1 + eps_minus_high)   if advantage < 0  (incorrect)
}
```

The paper's recommended values are: `eps_plus_low=0.2, eps_plus_high=10, eps_minus_low=1, eps_minus_high=100`. The stop-gradient operator `sg()` is applied to the clipped weights so gradients flow only through the log-probability term, making this a true REINFORCE-style update rather than a PPO-style surrogate objective.

## Step-by-Step Workflow

### 1. Define the reward function with verifiable correctness

Implement a binary reward based on answer verification (e.g., exact match against ground truth for math problems). Compute group-relative advantages:

```python
def compute_advantages(rewards, group_size):
    """Normalize rewards within each question's sample group."""
    # rewards shape: (num_questions, group_size)
    mu = rewards.mean(dim=-1, keepdim=True)
    sigma = rewards.std(dim=-1, keepdim=True).clamp(min=1e-8)
    advantages = (rewards - mu) / sigma  # shape: (num_questions, group_size)
    return advantages
```

### 2. Compute token-level importance sampling ratios

For each token in each response, compute the ratio between the current policy and the reference (frozen snapshot) policy:

```python
def compute_is_ratios(logprobs_current, logprobs_ref):
    """Token-level importance sampling ratios."""
    # Both shapes: (batch, seq_len)
    log_ratios = logprobs_current - logprobs_ref
    ratios = torch.exp(log_ratios)  # r_i,t(theta)
    return ratios
```

### 3. Apply decoupled clipping based on advantage sign

This is the core DISPO operation -- use different clipping bounds for positive vs. negative advantages:

```python
def dispo_clip(ratios, advantages, config):
    """Decoupled importance sampling weight clipping."""
    positive_mask = (advantages > 0).unsqueeze(-1)  # correct responses
    negative_mask = (advantages < 0).unsqueeze(-1)  # incorrect responses

    clipped_pos = torch.clamp(
        ratios,
        min=1.0 - config.eps_plus_low,    # default: 0.8
        max=1.0 + config.eps_plus_high     # default: 11.0
    )
    clipped_neg = torch.clamp(
        ratios,
        min=1.0 - config.eps_minus_low,   # default: 0.0
        max=1.0 + config.eps_minus_high    # default: 101.0
    )

    clipped = torch.where(positive_mask, clipped_pos, clipped_neg)
    return clipped.detach()  # stop-gradient: sg(r^d)
```

### 4. Compute the DISPO loss with token-level normalization

Apply length normalization (sum of token counts) rather than group-level averaging to prevent long-sequence bias:

```python
def dispo_loss(logprobs, clipped_weights, advantages, response_lengths):
    """DISPO policy gradient loss."""
    # advantages broadcast to token level
    token_advantages = advantages.unsqueeze(-1).expand_as(logprobs)
    per_token_loss = clipped_weights * token_advantages * logprobs

    # Mask padding, then normalize by total token count
    total_tokens = response_lengths.sum()
    loss = -per_token_loss.sum() / total_tokens
    return loss
```

### 5. (Optional) Add overlong response penalty

Discourage excessively long responses that waste compute without improving correctness:

```python
def overlong_penalty(response_lengths, max_length, penalty_weight=0.01):
    """Penalize responses exceeding target length."""
    excess = (response_lengths - max_length).clamp(min=0).float()
    return penalty_weight * excess.mean()
```

### 6. Filter uninformative training groups

Skip question groups where all responses are correct or all are incorrect (zero variance in advantages), as these provide no learning signal:

```python
def filter_groups(rewards, group_size):
    """Remove groups with uniform outcomes."""
    # rewards shape: (num_questions, group_size)
    has_correct = rewards.sum(dim=-1) > 0
    has_incorrect = rewards.sum(dim=-1) < group_size
    informative = has_correct & has_incorrect
    return informative
```

### 7. Set up the training loop with periodic reference policy updates

Freeze a reference policy snapshot and update it periodically. Sample multiple responses per question (G=16 in ablations):

```python
# Training loop skeleton
ref_policy = copy.deepcopy(model)
ref_policy.eval()

for step, batch in enumerate(dataloader):
    questions, ground_truths = batch
    # Sample G responses per question from current policy
    responses = model.generate(questions, num_return_sequences=G, do_sample=True)
    rewards = verify_answers(responses, ground_truths)  # binary: 0 or 1

    # Filter, compute advantages, IS ratios, clip, compute loss
    advantages = compute_advantages(rewards, G)
    mask = filter_groups(rewards, G)
    ratios = compute_is_ratios(model, ref_policy, questions, responses)
    clipped = dispo_clip(ratios, advantages[mask], config)
    loss = dispo_loss(logprobs[mask], clipped, advantages[mask], lengths[mask])

    loss.backward()
    optimizer.step()
    optimizer.zero_grad()

    if step % ref_update_interval == 0:
        ref_policy.load_state_dict(model.state_dict())
```

### 8. Monitor entropy and response length for early collapse detection

Track average token entropy and mean response length every N steps. Sudden drops in either signal an impending collapse:

```python
def monitor_health(logprobs, response_lengths, step, logger):
    """Log training health metrics."""
    entropy = -(logprobs.exp() * logprobs).sum(-1).mean()
    avg_length = response_lengths.float().mean()
    logger.log({"entropy": entropy.item(), "avg_length": avg_length.item()}, step=step)
    # Alert conditions
    if avg_length < 10:
        logger.warning(f"Step {step}: Length collapse detected (avg={avg_length:.1f})")
    if entropy < 0.5:
        logger.warning(f"Step {step}: Entropy collapse detected (H={entropy:.2f})")
```

### 9. Tune the four clipping parameters via targeted ablation

Start from the paper's defaults and adjust based on observed behavior:

| Symptom | Cause | Fix |
|---------|-------|-----|
| Accuracy degrades gradually after peak | `eps_plus_high` too large (over-exploration) | Reduce from 10 toward 5 |
| Model outputs repetitive sequences | `eps_minus_high` too small (weak unlearning) | Increase toward 100+ |
| Response lengths collapse to near-zero | `eps_minus_low` too small (over-correction) | Increase from 0 toward 1 |
| Entropy too low, model too deterministic | `eps_plus_low` too large (over-distillation) | Reduce from 0.2 toward 0.1 |

## Concrete Examples

**Example 1: Converting a GRPO training script to DISPO**

User: "I have a GRPO training loop for Qwen-7B on math data. It's stable but learning too slowly. Can you convert it to DISPO?"

Approach:
1. Identify the PPO-style surrogate objective with `min(r*A, clip(r, 1-eps, 1+eps)*A)` and remove it
2. Replace with REINFORCE-style: `sg(clip_decoupled(r)) * A * log_pi`
3. Split the single `eps=0.2` into four parameters: `eps_plus_low=0.2, eps_plus_high=10, eps_minus_low=1, eps_minus_high=100`
4. Apply stop-gradient to the clipped IS weights
5. Switch from group-level loss averaging to token-level normalization (`1/sum(|o_i|)`)
6. Add dynamic group filtering to skip all-correct or all-incorrect groups

Output: Modified training script with the decoupled clipping function, updated loss computation, and recommended hyperparameters as starting points.

**Example 2: Diagnosing repetition collapse in an existing RL setup**

User: "My CISPO-trained model started producing repetitive gibberish after 2000 steps. How do I fix this?"

Approach:
1. Identify the likely cause: uniform clipping with `eps_high=100` applies the same amplification to incorrect responses, but `eps_low` may be 0 or too small for incorrect responses
2. Check if Regime 3 (`eps_minus_high`) is effectively disabled -- if the incorrect-response up-clipping is capped too low, the model cannot unlearn confident mistakes
3. Migrate to DISPO's decoupled clipping, setting `eps_minus_high=100` to allow strong unlearning of incorrect tokens
4. Set `eps_minus_low=1` to prevent over-correction on already-unlikely incorrect tokens
5. Add entropy and length monitoring to catch future collapses early

Output: Diagnostic checklist, the specific parameter change, and monitoring code to prevent recurrence.

**Example 3: Implementing DISPO from scratch for a custom math reasoning dataset**

User: "I want to RL fine-tune Llama-3-8B on competition math problems using DISPO. Help me set up the full pipeline."

Approach:
1. Set up the verifiable reward function: parse model outputs for final numerical answers, compare against ground truth with exact match
2. Implement group sampling: for each question, generate G=16 candidate responses with temperature sampling
3. Implement group-relative advantage normalization: `A_i = (R_i - mean(R_group)) / std(R_group)`
4. Implement the decoupled clipping function with four independent epsilon parameters
5. Build the REINFORCE loss with stop-gradient weights and token-level normalization
6. Add dynamic filtering to skip uninformative groups
7. Add overlong response penalty
8. Set up monitoring for entropy, response length, and accuracy on a held-out validation set
9. Configure hyperparameters: `eps_plus_low=0.2, eps_plus_high=10, eps_minus_low=1, eps_minus_high=100`, lr=1e-6, G=16

Output: Complete training script with all components, configuration dataclass, and a monitoring dashboard setup.

## Best Practices

- **Do:** Start with the paper's default clipping values (`0.2, 10, 1, 100`) and only adjust after observing training dynamics for at least several hundred steps
- **Do:** Always apply `stop-gradient` (`.detach()` in PyTorch) to the clipped IS weights -- without this, DISPO degenerates into a PPO-style surrogate and loses its learning speed advantage
- **Do:** Use token-level length normalization (`1/sum(|o_i|)`) rather than group-level averaging to prevent the loss from being dominated by long responses
- **Do:** Filter out uninformative groups (all-correct or all-incorrect) before computing the loss, as these have zero-variance advantages and contribute noise
- **Avoid:** Setting `eps_minus_high` to 0 or very small values -- this disables Regime 3 (unlearning) and reliably causes repetition collapse
- **Avoid:** Setting `eps_minus_low` to 0 -- this disables Regime 4 (over-correction guard) and causes length collapse where the model learns to produce empty responses
- **Avoid:** Using `eps_plus_high` larger than ~10 without careful monitoring -- excessive exploration causes gradual accuracy degradation after an initial peak

## Error Handling

**Repetition collapse (sudden):** If the model starts generating repeated tokens or phrases, immediately check `eps_minus_high`. This value must be large enough (>=10, ideally 100) to allow the policy to strongly penalize high-probability incorrect tokens. As a temporary mitigation, apply a repetition penalty at inference time while retraining with corrected parameters.

**Length collapse (sudden):** If mean response length drops below ~10 tokens, check `eps_minus_low`. When this value is too close to 0, the clipping window `[1 - eps_minus_low, ...]` allows the IS weight to approach 0, creating unbounded negative gradients on incorrect tokens. Set `eps_minus_low >= 0.5` (paper uses 1.0).

**Gradual accuracy degradation:** If validation accuracy peaks and then steadily declines, `eps_plus_high` is too large, causing excessive exploration that destabilizes learned reasoning chains. Reduce it (e.g., from 10 to 5) and consider whether to restart from a checkpoint near peak accuracy.

**NaN losses:** Large IS ratios can cause numerical overflow. Clamp raw ratios to a safe range (e.g., `[1e-6, 1e4]`) before applying the decoupled clipping, and use mixed-precision training carefully (IS ratio computation should stay in fp32).

**Zero-variance groups dominate:** If most question groups are all-correct or all-incorrect, the effective batch size after filtering becomes very small. Either increase the number of samples per group (G), increase the difficulty of training questions, or use a curriculum that maintains ~30-70% solve rates.

## Limitations

- DISPO is designed for tasks with **verifiable binary rewards** (correct/incorrect). It does not directly apply to open-ended generation tasks without a clear verification function (e.g., creative writing, summarization).
- The four clipping parameters interact non-trivially; optimal values depend on model size, dataset difficulty, and training stage. The paper's defaults are a strong starting point but may require tuning for significantly different setups.
- Like all on-policy RL methods, DISPO requires generating multiple samples per question at each training step, which is computationally expensive (G=16 means 16x the inference cost per question).
- The technique was validated on mathematical reasoning benchmarks. Its effectiveness on other reasoning domains (code generation, logical reasoning, science) is plausible but not empirically confirmed in this paper.
- DISPO removes the trust-region guarantee of PPO-style methods. While the decoupled clipping prevents the catastrophic failures seen in CISPO, extremely aggressive parameter choices can still destabilize training.

## Reference

**Paper:** [DISPO: Enhancing Training Efficiency and Stability in Reinforcement Learning for Large Language Model Mathematical Reasoning](https://arxiv.org/abs/2602.00983v1) (Karaman et al., AISTATS 2026)

**What to look for:** Section 3 for the decoupled clipping formulation and the four-regime decomposition; Section 4 for the targeted ablation studies showing exactly how each regime affects entropy, response length, and accuracy; Table 1 for the recommended hyperparameter settings; Figures 3-5 for visual signatures of repetition collapse, length collapse, and gradual degradation.