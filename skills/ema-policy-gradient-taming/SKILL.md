---
name: "ema-policy-gradient-taming"
description: "Implement EMA Policy Gradient (EMA-PG) for stabilizing reinforcement learning fine-tuning of LLMs. Combines an Exponential Moving Average anchor policy with a Top-k KL estimator to improve GRPO and other policy gradient methods. Use when: 'stabilize RL training for LLM', 'implement EMA anchor for policy gradient', 'add top-k KL penalty to GRPO', 'tame reward hacking in RLHF', 'improve agentic RL training loop', 'set up EMA-PG with VeRL'."
---

# EMA Policy Gradient: Stabilizing RL for LLMs

This skill enables Claude to implement and configure the EMA-PG technique from Zhang & Ba (2026), which stabilizes reinforcement learning fine-tuning of large language models through two complementary mechanisms: (1) replacing the frozen reference policy with an Exponential Moving Average (EMA) anchor that slowly tracks the training policy, and (2) a Top-k KL divergence estimator that computes exact KL on the highest-probability tokens and corrects with sampled KL on the tail. Together, these yield large improvements over vanilla GRPO on both math reasoning (+3.1% OlympiadBench) and agentic search tasks (+33.3% average across 7 QA benchmarks).

## When to Use

- When the user is implementing or modifying a GRPO/PPO training loop for LLM post-training and wants to stabilize it against reward hacking or policy collapse.
- When the user asks how to add an EMA target network (like in DQN) to an LLM policy gradient setup.
- When the user needs a memory-efficient KL divergence computation for large-vocabulary models during RL training.
- When the user is building an agentic RL pipeline (e.g., LLM + search engine) and observes training instability or KL divergence explosion.
- When the user wants to integrate EMA-PG into a VeRL-based or custom RL training framework.
- When the user asks about reducing the gap between sampled and exact KL estimation in policy gradient methods.

## Key Technique

**EMA Anchor Policy.** Standard RLHF/GRPO uses a frozen copy of the initial supervised fine-tuned (SFT) model as the reference policy for KL regularization. This creates a growing mismatch: as the training policy improves, the static anchor becomes increasingly irrelevant, leading to either over-regularization (policy can't learn) or under-regularization (reward hacking). EMA-PG replaces this with a slowly-moving anchor updated as: `theta_ema = tau * theta_ema + (1 - tau) * theta_current`, where `tau` is a decay factor close to 1 (typically 0.9-0.999). This is directly analogous to the target network in deep Q-learning. The stability condition requires that `tau` be large enough that the KL between the anchor and training policy remains bounded -- concretely, the drift per step must be smaller than the regularization pull-back.

**Top-k KL Estimator.** Computing exact KL divergence over the full vocabulary (32k-150k tokens) at every training token position is expensive and requires materializing full logits for both the policy and anchor. The Top-k KL estimator identifies the `k` token indices with the largest probability mass under the current policy, computes exact KL on those indices, and uses importance-weighted sampled KL for the remaining tail. This provides unbiased KL values and unbiased gradients at any `k`, while concentrating computation where it matters most. Setting `k` to the full vocabulary recovers exact KL; setting `k=1` approximates sampled KL. In practice, `k=32` captures most of the KL mass for typical LLM distributions.

**Integration with GRPO.** The two techniques slot directly into the GRPO loss: the KL penalty term `beta * KL(pi || pi_ref)` simply uses the EMA model as `pi_ref` and the Top-k estimator for the KL computation. No changes to the reward computation, advantage estimation, or sampling procedure are needed.

## Step-by-Step Workflow

1. **Identify the existing RL training loop.** Locate the policy gradient loss computation, the reference model loading, and the KL penalty calculation in the codebase. In VeRL-based setups, this is typically in `core_algos.py` and the actor worker.

2. **Add EMA model state.** Create a copy of the policy model parameters to serve as the EMA anchor. Initialize it from the same checkpoint as the policy. Store it alongside the policy model (it shares the same architecture but has separate weights).

3. **Implement the EMA update rule.** After each training step (or every `N` steps), update the EMA anchor:
   ```python
   # theta_ema = tau * theta_ema + (1 - tau) * theta_policy
   for ema_param, policy_param in zip(ema_model.parameters(), policy_model.parameters()):
       ema_param.data.mul_(tau).add_(policy_param.data, alpha=1.0 - tau)
   ```
   Configure `tau` (e.g., 0.9) and `update_period` (e.g., every 10 steps).

4. **Replace frozen reference with EMA anchor.** Wherever the training loop computes KL against `ref_model`, redirect it to use `ema_model` instead. The EMA model should be in `eval()` mode and not receive gradients.

5. **Implement Top-k KL computation.** Write a function that:
   - Takes policy logits and EMA anchor logits at each token position.
   - Computes `log_probs = log_softmax(logits)` for both.
   - Selects the top-k indices from the policy distribution: `topk_indices = policy_log_probs.topk(k).indices`.
   - Gathers exact KL contributions on those k indices.
   - Estimates the tail KL via importance-weighted sampling from the remaining vocabulary.
   - Returns the combined KL estimate.

   ```python
   def compute_topk_kl(policy_logits, ema_logits, k=32):
       policy_lp = F.log_softmax(policy_logits, dim=-1)
       ema_lp = F.log_softmax(ema_logits, dim=-1)
       policy_p = policy_lp.exp()

       # Top-k exact KL
       topk_vals, topk_idx = policy_p.topk(k, dim=-1)
       topk_ema_lp = ema_lp.gather(-1, topk_idx)
       topk_policy_lp = policy_lp.gather(-1, topk_idx)
       kl_topk = (topk_vals * (topk_policy_lp - topk_ema_lp)).sum(dim=-1)

       # Tail correction via sampled KL
       tail_mass = 1.0 - topk_vals.sum(dim=-1, keepdim=True)
       kl_full_approx = (policy_p * (policy_lp - ema_lp)).sum(dim=-1)
       kl_tail = kl_full_approx - kl_topk  # unbiased correction
       # Or use importance-weighted sampling for memory efficiency

       return kl_topk + kl_tail
   ```

6. **Wire Top-k KL into the policy gradient loss.** Replace the existing KL penalty computation:
   ```python
   # Before: kl_penalty = compute_kl(policy_logits, ref_logits)
   # After:
   kl_penalty = compute_topk_kl(policy_logits, ema_logits, k=32)
   loss = -advantages * log_ratios + beta * kl_penalty
   ```

7. **Set hyperparameters.** Start with these values and tune:
   - `tau = 0.9` (EMA decay; higher = slower anchor movement)
   - `update_period = 10` (steps between EMA updates)
   - `k = 32` (top-k tokens for exact KL)
   - `beta = 0.01-0.1` (KL penalty coefficient)
   - `kl_loss_type = "full_reverse"` (KL direction: KL(pi || pi_ema))

8. **Validate stability.** Monitor the KL divergence between policy and EMA anchor during training. It should remain bounded (typically < 10 nats). If KL explodes, decrease the learning rate or increase `tau`. If KL collapses to near-zero, decrease `tau` or increase the learning rate.

9. **Test on a small-scale run.** Before committing to full training, run a few hundred steps on a subset of data and verify: (a) KL stays bounded, (b) reward increases, (c) EMA anchor parameters are actually updating.

10. **Scale up with VeRL integration.** For production runs, use the VeRL config flags:
    ```bash
    +algorithm.ref_policy_ema_tau=0.9
    +algorithm.ref_policy_ema_update_period=10
    actor_rollout_ref.actor.kl_loss_type=full_reverse
    actor_rollout_ref.actor.kl_topk_tokens=32
    actor_rollout_ref.actor.use_kl_iw=true
    ```

## Concrete Examples

**Example 1: Adding EMA-PG to an existing GRPO training script**

User: "I have a GRPO training loop for fine-tuning Qwen-2.5-3B on a tool-use dataset. Training keeps collapsing after ~500 steps. Can you stabilize it with EMA-PG?"

Approach:
1. Read the user's training script to locate the reference model and KL computation.
2. Add an EMA model initialized from the same checkpoint.
3. Replace the frozen reference KL with Top-k KL against the EMA anchor.
4. Set `tau=0.9`, `k=32`, `update_period=10`.
5. Add EMA parameter update after each optimizer step.

Output (key diff to the training loop):
```python
# --- Initialization ---
ema_model = deepcopy(policy_model)
ema_model.eval()
for p in ema_model.parameters():
    p.requires_grad = False

EMA_TAU = 0.9
EMA_UPDATE_PERIOD = 10
KL_TOPK = 32
KL_BETA = 0.04

# --- Inside training step ---
# Compute Top-k KL against EMA anchor (not frozen ref)
with torch.no_grad():
    ema_logits = ema_model(input_ids).logits
kl_penalty = compute_topk_kl(policy_logits, ema_logits, k=KL_TOPK)
loss = -grouped_advantages * log_ratios + KL_BETA * kl_penalty
loss.backward()
optimizer.step()

# Update EMA anchor periodically
if global_step % EMA_UPDATE_PERIOD == 0:
    for ema_p, pol_p in zip(ema_model.parameters(), policy_model.parameters()):
        ema_p.data.mul_(EMA_TAU).add_(pol_p.data, alpha=1.0 - EMA_TAU)
```

**Example 2: Configuring EMA-PG for agentic search RL with VeRL**

User: "I'm training a Qwen-3B model to use a search API for multi-hop QA using VeRL's GRPO. How do I enable EMA-PG?"

Approach:
1. Modify the VeRL training shell script to add EMA-PG flags.
2. Set appropriate hyperparameters for agentic tasks (longer rollouts need more conservative tau).
3. Ensure the training config enables importance-weighted KL.

Output (training script additions):
```bash
# train_grpo_ema_pg.sh
python -m verl.trainer.main_ppo \
    algorithm.ref_policy_ema_tau=0.9 \
    algorithm.ref_policy_ema_update_period=10 \
    actor_rollout_ref.actor.kl_loss_type=full_reverse \
    actor_rollout_ref.actor.kl_topk_tokens=32 \
    actor_rollout_ref.actor.use_kl_iw=true \
    actor_rollout_ref.actor.kl_use_tail_sampling=true \
    # ... rest of your existing config
```

**Example 3: Implementing memory-efficient Top-k KL for a custom trainer**

User: "My LLM has a 128k vocabulary and computing full KL at every token is blowing up GPU memory. How can I use Top-k KL?"

Approach:
1. Instead of materializing full softmax for both models, compute top-k indices from the policy model first.
2. Gather only those logits from the reference/EMA model.
3. Use importance-weighted tail correction.

Output:
```python
def compute_memory_efficient_topk_kl(policy_logits, ema_logits, k=32):
    """Memory-efficient: avoids full softmax materialization."""
    # Get top-k indices from policy logits directly (before softmax)
    _, topk_idx = policy_logits.topk(k, dim=-1)  # [B, T, k]

    # Gather only top-k logits from both models
    policy_topk = policy_logits.gather(-1, topk_idx)  # [B, T, k]
    ema_topk = ema_logits.gather(-1, topk_idx)        # [B, T, k]

    # Compute KL on top-k with local softmax
    policy_lp_topk = F.log_softmax(policy_topk, dim=-1)
    ema_lp_topk = F.log_softmax(ema_topk, dim=-1)
    policy_p_topk = policy_lp_topk.exp()

    kl = (policy_p_topk * (policy_lp_topk - ema_lp_topk)).sum(dim=-1)

    # Note: this is biased without tail correction.
    # For unbiased estimate, sample a few tail indices and apply
    # importance weighting -- see full implementation in repo.
    return kl
```

## Best Practices

- **Do:** Start with `tau=0.9` and `update_period=10` as a baseline, then tune. Higher tau (0.99+) is more conservative and suits unstable tasks; lower tau (0.5-0.9) lets the anchor track faster for stable settings.
- **Do:** Monitor the KL between policy and EMA anchor as a training diagnostic. Plot it alongside reward curves. Bounded, slowly-growing KL indicates healthy training.
- **Do:** Use `kl_loss_type=full_reverse` (KL(pi || pi_ema)) for the penalty direction -- this is mode-seeking and prevents the policy from spreading probability mass to low-quality outputs.
- **Do:** Keep `k` small (16-64) for memory savings on large vocabularies. The top tokens carry most of the KL mass in practice.
- **Avoid:** Setting `tau` too low (< 0.5) -- this makes the anchor nearly identical to the policy, eliminating the regularization effect.
- **Avoid:** Using EMA-PG without monitoring for KL collapse. If KL between policy and anchor drops to near zero, the anchor is tracking too fast and providing no regularization. Increase `tau` or decrease `update_period`.
- **Avoid:** Applying gradients through the EMA model. It must be in eval mode with `requires_grad=False` on all parameters.

## Error Handling

- **KL divergence explodes (> 50 nats).** The anchor is lagging too far behind. Increase `tau` (e.g., 0.9 -> 0.99), decrease the learning rate, or increase `update_period`. As an emergency measure, re-initialize the EMA anchor from the current policy checkpoint.
- **Training reward plateaus early.** The KL penalty may be too strong. Decrease `beta` by 2-5x, or decrease `tau` to let the anchor move faster, giving the policy more room to explore.
- **GPU OOM during KL computation.** Reduce `k` in the Top-k estimator (e.g., 32 -> 16). Use the memory-efficient gather-based implementation that avoids full vocabulary softmax.
- **EMA parameters not updating.** Verify the update logic runs outside `torch.no_grad()` for the parameter copy (but still without gradient tracking). Check that `update_period` isn't set unreasonably high.
- **NaN in KL values.** Usually caused by log(0) when probability mass vanishes. Add a small epsilon (`1e-8`) to probabilities before taking the log, or use `log_softmax` which is numerically stable.

## Limitations

- **Requires additional GPU memory.** The EMA anchor model is a full copy of the policy, roughly doubling the parameter memory footprint. For very large models, this may require model parallelism or offloading the EMA weights to CPU.
- **Hyperparameter sensitivity.** The optimal `tau` depends on the task, model size, and learning rate. There is no universal setting -- expect to tune on a per-task basis.
- **Not a replacement for reward engineering.** EMA-PG stabilizes optimization but cannot fix fundamentally misspecified reward functions. If the reward signal is noisy or misaligned, training will still struggle.
- **Assumes access to anchor logits.** The Top-k KL estimator requires a forward pass through the EMA model at each training step. This adds computational cost proportional to one extra inference pass per batch.
- **Tested primarily on Qwen models at 1.5B-3B scale.** Scaling behavior to 70B+ models is not yet empirically validated in the paper, though the method is architecturally agnostic.

## Reference

Zhang, L. & Ba, J. (2026). "EMA Policy Gradient: Taming Reinforcement Learning for LLMs with EMA Anchor and Top-k KL." arXiv:2602.04417v1. [https://arxiv.org/abs/2602.04417v1](https://arxiv.org/abs/2602.04417v1)

Look for: Section 3 (EMA anchor stability analysis), Section 4 (Top-k KL unbiasedness proofs), Algorithm 1 (full EMA-PG pseudocode), and Tables 1-3 (benchmark results).

Code: [https://github.com/LunjunZhang/ema-pg](https://github.com/LunjunZhang/ema-pg) -- see `search/verl/trainer/ppo/core_algos.py` for the Top-k KL implementation and EMA integration.