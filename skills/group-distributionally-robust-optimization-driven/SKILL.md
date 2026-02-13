---
name: "group-distributionally-robust-optimization-driven"
description: "Apply Group Distributionally Robust Optimization (GDRO) to RL-based LLM training. Dynamically classify prompts by difficulty, reweight training distributions toward hard examples, and allocate compute (rollouts) proportionally to gradient variance. Use when: 'optimize GRPO training for reasoning', 'apply GDRO to LLM post-training', 'adaptive difficulty curriculum for RL', 'reweight training prompts by hardness', 'allocate rollouts by difficulty group', 'improve RL training on heterogeneous reasoning data'."
---

# Group Distributionally Robust Optimization for RL-Based LLM Reasoning

This skill enables Claude to design, implement, and debug **Multi-Adversary GDRO** training pipelines for LLM reasoning. The core technique replaces GRPO's static uniform sampling and fixed rollout counts with two adversarial controllers: **Prompt-GDRO** (which upweights hard difficulty bins via EMA-debiased multiplicative weights) and **Rollout-GDRO** (which reallocates rollout budgets across bins via a shadow-price dual controller). Together they create an emergent curriculum that tracks the model's evolving reasoning frontier, yielding ~10% relative pass@8 gains at 1.7B--8B scale with no extra compute.

## When to Use

- When the user is implementing or modifying a GRPO/DAPO/PPO post-training loop and wants to add adaptive difficulty-aware reweighting
- When training data is heterogeneous (mix of easy and hard math/code/reasoning problems) and uniform sampling wastes compute on solved patterns
- When the user asks how to allocate more rollouts to harder prompts without increasing total compute
- When building a custom RL training loop and needing an online difficulty classifier with pass@k binning
- When the user wants to implement multiplicative-weights or shadow-price controllers inside a training loop
- When debugging training runs where easy problems dominate gradient signal and hard problems stagnate

## Key Technique

**The problem with uniform GRPO.** Standard Group Relative Policy Optimization samples prompts uniformly and generates a fixed number of rollouts per prompt. For heavy-tailed reasoning datasets, most compute is spent on prompts the model already solves, while genuinely hard prompts (the long tail) receive insufficient training signal. This structural inefficiency limits reasoning capability.

**GDRO's two-game solution.** GDRO introduces an Online Difficulty Classifier that buckets every prompt into one of B pass@k bins (e.g., 10 bins spanning [0, 0.1), [0.1, 0.2), ..., [0.9, 1.0)) using a sliding-window estimate of per-prompt success rate. Two independent adversarial games then operate over these bins. *Prompt-GDRO* maintains an EMA of mean loss per bin, converts these scores to exponential weights, mixes with a uniform exploration term, and reweights the GRPO advantage signal -- this upweights bins at the "intensive difficulty margin" (medium-hard problems where the model can still learn). *Rollout-GDRO* solves a constrained optimization: allocate per-bin rollout counts to maximize gradient variance reduction subject to a fixed mean-rollout budget, using a Lagrangian shadow-price update that converges to the square-root (Neyman) optimal allocation -- bins with higher intrinsic variance get more rollouts.

**Why it works.** Both controllers have no-regret guarantees, meaning the adversary converges to the worst-case distribution over bins. In practice this creates a "traveling wave" curriculum: as the model improves on medium-hard problems, those bins become easier, and the adversary shifts weight to the next frontier of difficulty. The framework is compute-neutral (same total rollouts as baseline) and composable (Prompt-GDRO and Rollout-GDRO can be used independently or together).

## Step-by-Step Workflow

1. **Define pass@k bins.** Partition the [0, 1] interval into B non-overlapping bins (typically B=10). Each bin represents a difficulty tier based on the model's empirical pass rate.

2. **Initialize per-prompt tracking.** For every unique prompt in the dataset, maintain a sliding-window buffer of length H storing binary pass/fail outcomes. Compute `pass@k_hat(x) = mean(buffer)` and assign prompt x to the bin whose interval contains this value. Add hysteresis margin delta (~0.05) to prevent thrashing.

3. **Implement the Prompt-GDRO adversary.** After each training step:
   - Compute mean loss per bin: `l_bar(b) = mean of per-prompt losses in bin b`
   - Update EMA score: `S(b) <- (1 - beta) * S(b) + beta * l_bar(b)` (beta ~ 0.1)
   - Compute unnormalized weights: `w(b) = exp(eta_q * clip(S(b), -C, C))` (eta_q ~ 1.0, C ~ 5.0)
   - Form sampling distribution: `q(b) = (1 - gamma) * w(b) / sum(w) + gamma / B` (gamma ~ 0.1)
   - Reweight GRPO advantages: `A_reweighted = A * min(w(bin(x)), w_max)`

4. **Implement the Rollout-GDRO controller.** Maintain dual variable mu (initialized to 0):
   - For each bin b, select rollout count `n_b` from {n_min, ..., n_max} that minimizes `v_b / n + mu * n` where v_b is estimated bin variance
   - Generate n_b rollouts for prompts in bin b
   - After the batch, update: `mu <- mu + alpha_mu * (n_realized - n_bar)` where n_bar is the budget and n_realized is actual mean rollouts used

5. **Estimate per-bin variance.** Track running variance of per-prompt GRPO loss within each bin. Use this as the v_b proxy for the Neyman allocation. Update with EMA to handle non-stationarity.

6. **Integrate with GRPO loss computation.** The standard GRPO clipped surrogate loss stays unchanged. Prompt-GDRO modifies only the advantage weighting. Rollout-GDRO modifies only the number of rollouts per prompt. Neither changes the loss function itself.

7. **Log and monitor adversary dynamics.** Track per-bin weights q(b), rollout allocations n_b, bin populations, and per-bin pass@k over training. Look for the "traveling wave" pattern: weight should shift from medium-hard bins toward harder bins as training progresses.

8. **Tune hyperparameters.** Key knobs are: eta_q (adversary sharpness -- too high concentrates on one bin, too low reverts to uniform), gamma (exploration rate), beta (EMA decay), alpha_mu (dual step size), and n_min/n_max (rollout bounds). Start with the paper's defaults and adjust based on bin weight entropy.

9. **Validate with pass@k evaluation.** Evaluate on held-out prompts using pass@8 (or pass@k for your k). Compare against a GRPO baseline with uniform sampling and fixed rollouts. Expect ~10% relative improvement on heterogeneous reasoning benchmarks.

## Concrete Examples

**Example 1: Adding Prompt-GDRO to an existing GRPO training script**

User: "I have a GRPO training loop for math reasoning. Easy problems dominate training and hard problems aren't improving. How do I add difficulty-aware reweighting?"

Approach:
1. Add a prompt-level pass@k tracker using a sliding window of the last H=20 evaluations per prompt
2. Assign each prompt to one of 10 difficulty bins based on its empirical pass rate
3. After computing GRPO advantages, reweight them by the Prompt-GDRO multiplicative weights

```python
import numpy as np
from collections import defaultdict, deque

class PromptGDRO:
    def __init__(self, n_bins=10, eta_q=1.0, beta=0.1, gamma=0.1,
                 clip_C=5.0, w_max=10.0, window_H=20):
        self.n_bins = n_bins
        self.eta_q = eta_q
        self.beta = beta
        self.gamma = gamma
        self.clip_C = clip_C
        self.w_max = w_max
        self.window_H = window_H
        # Per-prompt pass history
        self.pass_history = defaultdict(lambda: deque(maxlen=window_H))
        # Per-bin EMA score
        self.S = np.zeros(n_bins)

    def get_bin(self, prompt_id: str) -> int:
        history = self.pass_history[prompt_id]
        if len(history) == 0:
            return self.n_bins // 2  # default to middle bin
        pass_rate = np.mean(list(history))
        return min(int(pass_rate * self.n_bins), self.n_bins - 1)

    def update_pass(self, prompt_id: str, any_correct: bool):
        self.pass_history[prompt_id].append(float(any_correct))

    def compute_weights(self, bin_losses: dict) -> np.ndarray:
        """bin_losses: {bin_id: mean_loss} for bins seen this step."""
        for b, loss in bin_losses.items():
            self.S[b] = (1 - self.beta) * self.S[b] + self.beta * loss
        clipped = np.clip(self.S, -self.clip_C, self.clip_C)
        w = np.exp(self.eta_q * clipped)
        q = (1 - self.gamma) * w / w.sum() + self.gamma / self.n_bins
        return q

    def reweight_advantage(self, advantage: float, prompt_id: str) -> float:
        b = self.get_bin(prompt_id)
        clipped = np.clip(self.S[b], -self.clip_C, self.clip_C)
        weight = min(np.exp(self.eta_q * clipped), self.w_max)
        return advantage * weight
```

Output: Advantages for prompts in hard bins (low pass rate, high loss) get amplified, while easy bins get downweighted. The EMA prevents stale scores from dominating.

---

**Example 2: Implementing Rollout-GDRO for compute-neutral variable rollouts**

User: "I want to give harder prompts more rollouts without increasing total compute. How do I implement the shadow-price controller?"

Approach:
1. Track per-bin gradient variance as a proxy for how much each bin benefits from extra rollouts
2. Use a Lagrangian dual variable to enforce the mean-rollout budget constraint
3. Solve the per-bin allocation using the square-root rule as initialization

```python
class RolloutGDRO:
    def __init__(self, n_bins=10, n_bar=4, n_min=2, n_max=12,
                 alpha_mu=0.05):
        self.n_bins = n_bins
        self.n_bar = n_bar          # mean rollout budget
        self.n_min = n_min
        self.n_max = n_max
        self.alpha_mu = alpha_mu
        self.mu = 0.0               # shadow price (dual variable)
        self.var_ema = np.ones(n_bins)  # per-bin variance estimates

    def update_variance(self, bin_id: int, losses: list, beta=0.1):
        """Update variance estimate for a bin from per-prompt losses."""
        if len(losses) > 1:
            v = np.var(losses)
            self.var_ema[bin_id] = (1 - beta) * self.var_ema[bin_id] + beta * v

    def allocate_rollouts(self, bin_fractions: np.ndarray) -> np.ndarray:
        """Return rollout count per bin. bin_fractions: fraction of prompts in each bin."""
        # Square-root (Neyman) allocation
        sqrt_v = np.sqrt(self.var_ema + 1e-8)
        n_star = self.n_bar * sqrt_v / (bin_fractions @ sqrt_v + 1e-8)
        # Clamp to [n_min, n_max]
        n_alloc = np.clip(np.round(n_star), self.n_min, self.n_max).astype(int)
        return n_alloc

    def update_dual(self, n_realized_mean: float):
        """Update shadow price after observing actual mean rollouts used."""
        self.mu += self.alpha_mu * (n_realized_mean - self.n_bar)
        self.mu = max(self.mu, 0.0)  # shadow price is non-negative
```

Output: Hard bins (high variance) receive up to n_max rollouts; easy bins drop to n_min. Total mean stays at n_bar. The dual variable mu self-corrects if budget drifts.

---

**Example 3: Diagnosing a stalled GDRO training run**

User: "My GDRO training shows all weight concentrated in the hardest bin and training is unstable."

Approach:
1. Check eta_q -- if too high, the adversary becomes too aggressive. Reduce from 1.0 to 0.3
2. Check gamma -- exploration mixture should be at least 0.1 to prevent degenerate concentration
3. Check bin populations -- if the hardest bin has very few prompts, high weight on sparse data causes noisy gradients
4. Verify w_max clipping is active to prevent extreme advantage scaling
5. Look at the EMA decay beta -- if too high (close to 1), scores are too reactive to single-batch noise

Output: Reduce eta_q to 0.3, ensure gamma >= 0.1, set w_max = 5.0. If the hardest bin has < 5% of prompts, merge the top two bins. Training should stabilize within 50 steps.

## Best Practices

- **Do** start with Prompt-GDRO alone before adding Rollout-GDRO. Prompt-GDRO is simpler and provides most of the gain.
- **Do** use hysteresis (delta ~ 0.05) in bin assignment to prevent prompts from oscillating between bins every step.
- **Do** log bin weight entropy over training. Healthy training shows entropy decreasing gradually (not collapsing to zero).
- **Do** use the EMA-debiased scoring (the beta parameter) rather than raw batch statistics, which are too noisy for small batch sizes.
- **Avoid** setting eta_q > 2.0 -- this over-concentrates on a single bin and destabilizes training.
- **Avoid** using Rollout-GDRO with n_max / n_min ratio > 6. Extreme rollout imbalance causes uneven batch processing and GPU utilization issues.
- **Avoid** applying GDRO weights to the KL penalty term in GRPO. Only reweight the advantage/reward signal, not the regularization.

## Error Handling

| Problem | Likely Cause | Fix |
|---|---|---|
| All weight collapses to one bin | eta_q too high or gamma too low | Reduce eta_q, increase gamma to 0.2 |
| Weights are nearly uniform (no effect) | eta_q too low or beta too high (over-smoothed) | Increase eta_q to 1.0, reduce beta to 0.05 |
| Rollout budget consistently exceeded | alpha_mu too low, dual variable adapts too slowly | Increase alpha_mu by 2x, or tighten n_max |
| Empty bins causing NaN in loss averaging | Some difficulty levels have zero prompts | Skip empty bins in score updates; use Laplace smoothing for bin fractions |
| Pass@k estimates are noisy early in training | Small sliding window H | Increase H to 30-50, or initialize all prompts to the middle bin |
| Training diverges after adding GDRO | w_max too high allowing extreme advantage scaling | Cap w_max at 5.0; verify gradient norms are stable |

## Limitations

- **Requires per-prompt identity tracking.** The sliding-window pass@k estimator needs a unique ID for each prompt. Datasets with heavy augmentation or on-the-fly generation break this assumption.
- **Bin boundaries are a hyperparameter.** 10 uniform bins work well for math reasoning, but other domains (code, long-form QA) may need different granularity or non-uniform spacing.
- **Cold start.** Until each prompt has been seen H times, pass@k estimates are unreliable. The first ~H epochs effectively run uniform GRPO.
- **Single-task assumption.** GDRO groups by difficulty within a single task type. For multi-task training (math + code + language), you need separate bin structures per task or a multi-dimensional grouping scheme.
- **Diminishing returns at very large scale.** The paper validates up to 8B parameters. At much larger scales, the baseline GRPO may already handle heterogeneity better, reducing GDRO's relative advantage.

## Reference

**Paper:** [Group Distributionally Robust Optimization-Driven Reinforcement Learning for LLM Reasoning](https://arxiv.org/abs/2601.19280v1) (Panaganti et al., 2026). Look for: Algorithm 1 (Prompt-GDRO), Algorithm 2 (Rollout-GDRO), Theorem 3.1 (no-regret bound), Proposition 3.2 (square-root allocation), and Figure 3 (traveling wave visualization).