---
name: "rethinking-trust-region-reinforcement"
description: "Implement Divergence Proximal Policy Optimization (DPPO) for LLM reinforcement learning fine-tuning, replacing PPO's ratio clipping with principled divergence constraints. Use when: 'implement DPPO for LLM training', 'fix PPO instability in RLHF', 'replace ratio clipping with divergence constraint', 'stable RL fine-tuning for large vocabulary models', 'implement trust region for LLM RL', 'DPPO with binary or top-k approximation'."
---

# Divergence Proximal Policy Optimization (DPPO) for LLM RL Fine-Tuning

This skill enables Claude to implement DPPO — a replacement for PPO's ratio clipping mechanism in LLM reinforcement learning. DPPO substitutes the heuristic `clip(r_t, 1-eps, 1+eps)` with a direct divergence constraint (Total Variation or KL) between the rollout and updated policies. It uses efficient Binary or Top-K approximations to estimate divergence without materializing full vocabulary distributions, solving PPO's fundamental problem: over-penalizing low-probability token updates while under-constraining high-probability token shifts.

## When to Use

- When the user is implementing or debugging PPO-based RLHF/RL training for LLMs and encounters instability (reward collapse, training spikes, divergence)
- When building a reinforcement learning training loop (GRPO, PPO, REINFORCE variants) for language models and needs a stable trust region mechanism
- When the user asks to replace ratio clipping in their RL code with a divergence-based constraint
- When implementing reward-based fine-tuning (math reasoning, code generation, instruction following) and the model has a large vocabulary (32K-150K+ tokens)
- When the user wants to reduce training-inference mismatch in RL-tuned LLMs
- When adapting policy gradient methods from small action spaces to the massive discrete action spaces of language models

## Key Technique

**The Problem with PPO Ratio Clipping in LLMs.** Standard PPO clips the importance sampling ratio `r_t = pi(a_t|s_t) / mu(a_t|s_t)` to `[1-eps, 1+eps]`. This ratio is a single-sample Monte Carlo estimate of policy divergence — fine for small action spaces but structurally broken for vocabularies of 100K+ tokens. Consider: a token with `mu=0.0001` moving to `pi=0.01` produces `r_t=100` (aggressively clipped), yet the actual TV divergence is only ~0.01. Meanwhile, a dominant token shifting from `mu=0.99` to `pi=0.80` gives `r_t=0.808` (unclipped), despite redistributing 0.19 probability mass. PPO over-penalizes rare token exploration and under-constrains catastrophic shifts in common tokens.

**DPPO: Divergence-Based Masking.** DPPO replaces clipping with a divergence-aware mask. The objective is `L_DPPO = E[M_t * r_t * A_t]`, where the mask `M_t` is 0 (blocks the update) when three conditions hold simultaneously: (1) the advantage direction matches the ratio direction (positive advantage with r_t > 1, or negative advantage with r_t < 1), (2) the ratio is already moving away from the reference policy, and (3) the estimated divergence `D(mu || pi)` at that token position exceeds a threshold `delta`. This means DPPO only intervenes when the update is expanding the trust region boundary — it never blocks updates that move the policy back toward the reference.

**Efficient Approximations.** Computing full KL or TV over 100K+ tokens at every position is prohibitive. The **Binary approximation** collapses the vocabulary into a Bernoulli: sampled-token vs. everything-else. `D_TV_Bin = |mu(a_t|s_t) - pi(a_t|s_t)|` and `D_KL_Bin = mu*log(mu/pi) + (1-mu)*log((1-mu)/(1-pi))`. The **Top-K approximation** selects the K highest-probability tokens under `mu`, unions in the sampled token, aggregates the rest into an "other" bucket, then computes divergence over this reduced set. In practice, K=20 works well and Binary alone is often sufficient.

## Step-by-Step Workflow

1. **Identify the PPO clipping code** in the existing training loop. Look for the characteristic pattern: `torch.clamp(ratio, 1-eps, 1+eps)` or `torch.min(ratio * advantage, clipped_ratio * advantage)`. This is what DPPO replaces.

2. **Preserve rollout log-probabilities.** During trajectory generation, store `mu_logprobs[t]` — the log-probability of each sampled token under the rollout policy. Also store `mu_probs_at_token[t] = exp(mu_logprobs[t])` for the sampled token. For Top-K, additionally store the top-K token indices and their probabilities from the rollout policy at each position.

3. **Compute current policy probabilities at update time.** During PPO epochs, compute `pi_probs_at_token[t]` for the sampled token. For Binary approximation, this is all you need. For Top-K, also gather `pi_probs` at the stored top-K indices.

4. **Compute the divergence estimate.** Choose Binary (simpler, sufficient for most cases) or Top-K (more accurate):
   - **Binary TV:** `D = |mu_prob - pi_prob|`
   - **Binary KL:** `D = mu_prob * log(mu_prob / pi_prob) + (1 - mu_prob) * log((1 - mu_prob) / (1 - pi_prob))`
   - **Top-K TV:** Aggregate remaining mass into "other", compute `0.5 * sum(|mu_k - pi_k|)` over K+1 buckets
   - **Top-K KL:** Same aggregation, compute `sum(mu_k * log(mu_k / pi_k))` over K+1 buckets

5. **Build the DPPO mask.** Compute `M_t` as a binary tensor:
   ```python
   expanding = (advantage > 0) & (ratio > 1) | (advantage < 0) & (ratio < 1)
   mask = ~(expanding & (divergence > delta))  # 1 where update is allowed
   ```
   The mask is 1 (allow) by default; it only blocks updates that are simultaneously expanding the trust region AND the divergence already exceeds the threshold.

6. **Compute the DPPO loss.** Replace the PPO clipped objective with:
   ```python
   loss = -(mask.float() * ratio * advantage).mean()
   ```
   No min/clip operations. The mask handles trust region enforcement.

7. **Set the divergence threshold `delta`.** Start with `delta=0.2` for TV or `delta=0.1` for KL. Tune based on training stability — lower values are more conservative. Unlike PPO's epsilon, this threshold directly corresponds to meaningful distributional shift.

8. **Optionally apply directional relaxation for low-probability tokens.** For tokens where `mu(a_t|s_t) < alpha` (e.g., `alpha=0.1`), relax the mask in both directions (Relax-Both). This prevents DPPO from inheriting PPO's over-penalization of rare token exploration:
   ```python
   low_prob = mu_probs_at_token < alpha
   mask[low_prob] = 1  # always allow updates on low-prob tokens
   ```

9. **Anchor the trust region to the rollout policy, not the recomputed policy.** Always measure divergence against `mu` (the policy that generated the trajectories), never against `pi` re-evaluated at the start of each epoch. Using recomputed probabilities decouples the constraint from the actual data distribution and causes training collapse.

10. **Monitor training-inference mismatch.** Track the gap between rewards measured during rollout vs. rewards on a held-out set evaluated greedily. DPPO should maintain mismatch below ~5%. If mismatch grows, decrease `delta`.

## Concrete Examples

**Example 1: Replacing PPO clipping in a GRPO math reasoning trainer**

User: "My GRPO training loop uses PPO clipping and I'm seeing reward spikes followed by collapse on GSM8K. Replace the clipping with DPPO."

Approach:
1. Locate the PPO loss computation in the training loop
2. Replace clipping with Binary TV divergence mask
3. Add divergence monitoring

Original PPO code:
```python
ratio = torch.exp(log_probs - old_log_probs)
clipped_ratio = torch.clamp(ratio, 1 - eps, 1 + eps)
loss = -torch.min(ratio * advantages, clipped_ratio * advantages).mean()
```

DPPO replacement:
```python
ratio = torch.exp(log_probs - old_log_probs)
mu_prob = torch.exp(old_log_probs)
pi_prob = torch.exp(log_probs)

# Binary TV divergence
divergence = torch.abs(mu_prob - pi_prob)

# DPPO mask: block only trust-region-expanding updates beyond threshold
expanding = ((advantages > 0) & (ratio > 1)) | ((advantages < 0) & (ratio < 1))
mask = ~(expanding & (divergence > delta))

# Optional: relax constraint for low-probability tokens
low_prob_tokens = mu_prob < 0.1
mask = mask | low_prob_tokens

loss = -(mask.float() * ratio * advantages).mean()
```

**Example 2: Implementing Top-K divergence for a 150K-token vocabulary model**

User: "I'm fine-tuning Qwen3-30B with RL on math. The vocabulary is huge. Implement DPPO with Top-K approximation."

Approach:
1. During rollout, store top-K indices and probabilities alongside sampled token logprobs
2. At update time, gather current policy probs at those indices
3. Compute Top-K TV divergence with "other" bucket

```python
# During rollout (store these alongside trajectories)
with torch.no_grad():
    logits = model(input_ids).logits[:, -1, :]
    probs = torch.softmax(logits, dim=-1)
    topk_probs, topk_indices = torch.topk(probs, k=20, dim=-1)
    # Store: topk_probs, topk_indices, sampled_token_id, sampled_token_prob

# During DPPO update
pi_logits = model(input_ids).logits[:, -1, :]
pi_probs_full = torch.softmax(pi_logits, dim=-1)

# Gather probs at stored top-K positions
pi_topk_probs = pi_probs_full.gather(-1, stored_topk_indices)

# "Other" bucket: 1 - sum(top-K probs)
mu_other = 1.0 - stored_topk_probs.sum(dim=-1, keepdim=True)
pi_other = 1.0 - pi_topk_probs.sum(dim=-1, keepdim=True)

# Concatenate with "other" → (batch, K+1)
mu_dist = torch.cat([stored_topk_probs, mu_other], dim=-1)
pi_dist = torch.cat([pi_topk_probs, pi_other], dim=-1)

# Top-K TV divergence
divergence = 0.5 * torch.abs(mu_dist - pi_dist).sum(dim=-1)

# Apply DPPO mask as in Example 1
```

**Example 3: Diagnosing PPO instability caused by ratio clipping**

User: "My PPO-based RLHF training diverges after 200 steps. How do I tell if the ratio clipping is the problem?"

Approach:
1. Log per-token ratio values and the corresponding token probabilities under `mu`
2. Check for the signature pathology: high clip rates on low-probability tokens, low clip rates on high-probability tokens despite large probability mass shifts

```python
# Add this diagnostic logging inside your PPO loop
with torch.no_grad():
    ratio = torch.exp(log_probs - old_log_probs)
    mu_prob = torch.exp(old_log_probs)
    clipped = (ratio < 1 - eps) | (ratio > 1 + eps)

    # Signature pathology: low-prob tokens clipped disproportionately
    low_prob_mask = mu_prob < 0.01
    high_prob_mask = mu_prob > 0.5

    low_prob_clip_rate = clipped[low_prob_mask].float().mean()
    high_prob_clip_rate = clipped[high_prob_mask].float().mean()

    # Also measure actual divergence on high-prob tokens
    high_prob_tv = torch.abs(mu_prob[high_prob_mask] - torch.exp(log_probs[high_prob_mask]))

    print(f"Low-prob clip rate: {low_prob_clip_rate:.3f}")   # Expect: high (>0.5)
    print(f"High-prob clip rate: {high_prob_clip_rate:.3f}")  # Expect: low (<0.1)
    print(f"High-prob mean TV: {high_prob_tv.mean():.4f}")    # Expect: significant
```

If you see high clip rates on low-prob tokens and large unconstrained divergence on high-prob tokens, switch to DPPO.

## Best Practices

- **Do:** Start with Binary TV approximation. It requires no extra stored tensors beyond what PPO already needs (old log-probs) and performs comparably to Top-K in practice.
- **Do:** Always anchor divergence to the rollout policy `mu` (the frozen snapshot that generated trajectories). Never recompute the reference at each PPO epoch.
- **Do:** Apply directional relaxation (Relax-Both) for tokens with `mu < 0.1`. This is critical for allowing the model to explore rare but potentially high-reward tokens.
- **Do:** Monitor the fraction of masked (blocked) updates per batch. If it exceeds ~30%, decrease `delta` — the policy is changing too aggressively.
- **Avoid:** Using truncated importance sampling (TIS) with DPPO. The paper shows TIS variants cause premature training collapse by zeroing out gradients for informative samples.
- **Avoid:** Setting K too large in Top-K approximation. K=20 captures the essential divergence; K=100+ wastes memory storing and gathering probabilities with minimal accuracy gain.
- **Avoid:** Mixing divergence-based masking with ratio clipping. Choose one mechanism. Combining them re-introduces the over-penalization problem on low-probability tokens.

## Error Handling

| Problem | Symptom | Fix |
|---------|---------|-----|
| Reward collapse after initial gains | Mean reward drops sharply, never recovers | Decrease `delta` (tighten trust region); verify divergence is anchored to rollout policy, not recomputed |
| Training-inference mismatch grows | Rollout rewards rise but eval rewards plateau or fall | The policy is overfitting to its own generation distribution. Decrease `delta` or add KL penalty to reference policy |
| NaN in divergence computation | Binary KL produces NaN/Inf | Clamp probabilities: `pi_prob = pi_prob.clamp(min=1e-8, max=1-1e-8)` before computing log ratios |
| Mask blocks nearly all updates | >50% updates masked | `delta` is too low or the learning rate is too high. Increase `delta` or reduce LR |
| No improvement over PPO | DPPO and PPO perform identically | Check that the low-probability relaxation is active. Without it, DPPO may be similarly conservative on rare tokens |

## Limitations

- DPPO addresses the trust region mechanism only. It does not fix issues with reward model quality, advantage estimation, or data distribution — those require separate solutions.
- Binary approximation discards information about non-sampled tokens. For tasks where multiple plausible continuations compete (creative writing, open-ended generation), Top-K may be necessary.
- The divergence threshold `delta` requires tuning per task and model scale. There is no universal default, though `delta=0.2` (TV) is a reasonable starting point.
- DPPO assumes the rollout policy is stored and available during updates. Fully online single-sample methods without a replay buffer cannot use this approach directly.
- The paper's experiments focus on mathematical reasoning benchmarks (GSM8K, MATH, AIME). Effectiveness on other RL objectives (helpfulness, harmlessness, code generation) is plausible but not directly demonstrated.

## Reference

**Paper:** [Rethinking the Trust Region in LLM Reinforcement Learning](https://arxiv.org/abs/2602.04879v1) (Qi et al., 2026). Key sections: Section 3 for the theoretical motivation and performance bounds; Section 4 for the DPPO objective and Binary/Top-K approximation formulas (Equations 11-16); Section 5 for the stability analysis showing that <0.5% of updates cause training instability and that anchoring to rollout policy is essential.