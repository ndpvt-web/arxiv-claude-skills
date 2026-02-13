---
name: "predicting-improving-test-time-scaling"
description: "Implement Scaling-Law Guided (SLG) Search for test-time compute optimization. Uses reward tail distribution estimation (GPD fitting) to predict scaling laws and dynamically allocate compute budget across candidate solutions. Trigger phrases: 'optimize test-time compute', 'best-of-N scaling', 'SLG search', 'tail-guided search', 'reward-guided budget allocation', 'test-time scaling law'"
---

# Predicting and Improving Test-Time Scaling via Reward Tail-Guided Search

This skill enables Claude to implement and apply Scaling-Law Guided (SLG) Search, a test-time compute optimization algorithm that replaces naive best-of-N sampling with principled budget allocation. Instead of uniformly sampling N candidates and picking the best, SLG Search fits a Generalized Pareto Distribution (GPD) to the tail of observed rewards, predicts how much improvement additional compute would yield per candidate path, and concentrates remaining budget on the most promising intermediate states. This achieves the same reward quality as best-of-N but with polynomially less compute.

## When to Use

- When the user wants to optimize how an LLM allocates test-time compute across multiple candidate solutions (e.g., math reasoning, code generation, planning)
- When building a best-of-N sampling pipeline and wanting principled guidance on how large N should be or how to split budget across stages
- When implementing a search algorithm over LLM outputs that needs to decide which partial solutions to expand further
- When the user asks to predict LLM scaling behavior without running exhaustive evaluations
- When designing multi-stage reasoning pipelines where intermediate states compete for compute budget
- When implementing reward-model-guided tree search and needing a principled node selection criterion beyond simple reward ranking

## Key Technique

**The Problem with Best-of-N:** Vanilla best-of-N generates N independent samples, scores them with a reward model, and returns the best. This is wasteful because it treats every sample path equally. Some intermediate states lead to reward distributions with heavy tails (high upside potential), while others have light tails (diminishing returns). BoN cannot distinguish between them.

**Tail-Guided Prediction:** SLG Search estimates the tail behavior of rewards at each intermediate state by fitting a Generalized Pareto Distribution to the upper quantile of observed reward samples. The GPD shape parameter (xi) captures whether the reward distribution is heavy-tailed (xi > 0, power-law decay, meaning more compute will likely find much better solutions) or light-tailed (xi near 0, exponential decay, meaning returns diminish quickly). From just m small pilot samples at each state, the algorithm predicts the expected maximum reward achievable with any budget N via the scaling law: V(s, N) ~ baseline + scale * N^(xi). This prediction avoids exhaustive evaluation.

**Dynamic Budget Allocation:** The algorithm follows an Expand-Estimate-Select-Exploit pipeline. First, expand K candidate intermediate states. Then estimate each state's tail index from m pilot rollouts. Select the state with the highest predicted V(s, N_remaining). Finally, exploit by concentrating all remaining compute on that state. The hyperparameters K and m are auto-derived from total budget N: m = max(20, log(N)^3 / 5) and K = N / (2m), ensuring enough pilot samples for reliable GPD fits while maximizing exploitation budget. This achieves vanishing regret versus an oracle that knows the true reward distributions.

## Step-by-Step Workflow

1. **Define the budget and objective.** Fix the total compute budget N (number of LLM rollouts allowed). Identify the reward signal: a reward model score, task accuracy, or any scalar quality metric over LLM outputs.

2. **Compute the hyperparameters K and m from N.** Set m = max(20, floor(log(N)^3 / 5)) as the pilot sample count per candidate state. Set K = floor(N / (2 * m)) as the number of intermediate states to explore. Ensure K >= 2 and K * m + K <= N (leaving budget for exploitation).

3. **Generate K intermediate states (Expand phase).** From the input prompt, use the LLM to generate K diverse partial solutions. For single-step tasks, these are K independent completions of the first portion of the response. For multi-step tasks, these are K different reasoning prefixes or first-step outputs.

4. **Pilot-sample each state (Estimate phase).** For each of the K intermediate states, generate m full rollout completions. Score each completion with the reward model. Collect the reward vectors {r_1, ..., r_m} for each state.

5. **Fit the GPD tail model to each state's rewards.** For each state, extract the top 20% of reward values (the tail). Fit a Generalized Pareto Distribution via maximum likelihood estimation (use `scipy.stats.genpareto.fit`). Record the shape parameter xi (tail index) and scale parameter sigma for each state.

6. **Predict scaling potential V(s, N_remaining).** For each state s with tail parameters (xi_s, sigma_s), estimate the expected best reward achievable if the remaining budget (N_remaining = N - K*m) were allocated entirely to s. Use: V(s, N_remaining) = quantile_mean_s + sigma_s / (1 - xi_s) * (N_remaining)^(xi_s) where quantile_mean_s is the mean of the top-20% rewards already observed. States with larger xi have more upside.

7. **Select the best state (Select phase).** Choose s* = argmax_s V(s, N_remaining). This is the intermediate state whose reward tail predicts the highest payoff from additional compute.

8. **Allocate remaining budget to s* (Exploit phase).** Generate N_remaining additional rollouts continuing from state s*. Score all outputs with the reward model.

9. **Return the global best.** Across all K*m pilot samples and N_remaining exploitation samples, return the single response with the highest reward score.

10. **Log and validate.** Record the predicted V(s*, N_remaining) versus the actual best reward achieved. Track tail index estimates across problems to calibrate future budget splits.

## Concrete Examples

**Example 1: Optimizing Math Problem Solving with Limited Compute**

```
User: I have a budget of 500 LLM calls per math problem. I'm currently using
best-of-500 but want better results. Implement SLG search.

Approach:
1. Compute hyperparameters: m = max(20, floor(log(500)^3/5)) = max(20, floor(238.3/5)) = 47.
   K = floor(500 / (2*47)) = 5. Exploitation budget = 500 - 5*47 = 265.
2. Generate 5 diverse first-step reasoning approaches for the math problem.
3. For each approach, generate 47 full solutions and score with reward model.
4. Fit GPD to top 20% rewards (top ~9 scores) for each of the 5 approaches.
5. Predict V(s, 265) for each approach using fitted tail parameters.
6. Suppose approach #3 has xi=0.15 (heavy tail) while others have xi~0.02.
   Approach #3 gets all 265 remaining rollouts.
7. Return the single best-scoring solution across all 500 rollouts.

Output:
- Best-of-500 (baseline): average reward 0.72 on AIME problems
- SLG Search (same budget): average reward 0.81 on AIME problems
- Effective equivalent BoN: ~2000 (4x compute savings)
```

**Example 2: Predicting Scaling Laws Without Exhaustive Evaluation**

```
User: I want to know how my model's performance scales from N=100 to N=10000
without actually running 10000 samples per problem. Predict the scaling curve.

Approach:
1. Run a small pilot: generate 100 samples per problem, score with reward model.
2. Fit GPD to the top 20 rewards (top 20%) for each problem.
3. Extract the average tail index xi across problems.
4. Predict the scaling curve: E[max reward at N] ~ mu + sigma * N^xi
   for N in [100, 200, 500, 1000, 2000, 5000, 10000].
5. Plot predicted curve. Validate at N=200 with actual samples if desired.

Output:
Predicted scaling curve (xi = 0.08 average):
  N=100:  0.65 reward  (observed)
  N=200:  0.69 reward  (predicted)
  N=500:  0.74 reward  (predicted)
  N=1000: 0.78 reward  (predicted)
  N=5000: 0.85 reward  (predicted)
  N=10000: 0.88 reward (predicted)

Interpretation: Light tail (xi=0.08) means diminishing returns.
Going from N=1000 to N=10000 (10x compute) gains only +0.10 reward.
Budget beyond N=1000 may not be cost-effective for this model.
```

**Example 3: Implementing SLG Search in Python**

```
User: Give me a minimal implementation of the SLG search core logic.

Output:
```python
import numpy as np
from scipy.stats import genpareto
from math import log, floor

def slg_search(prompt, generate_fn, score_fn, N=1000):
    """
    SLG Search: Scaling-Law Guided test-time compute allocation.

    Args:
        prompt: Input problem/prompt string
        generate_fn(prompt, prefix, n): generates n completions from prefix
        score_fn(completions): returns array of reward scores
        N: total compute budget (number of rollouts)
    Returns:
        best_completion, best_score
    """
    # Step 1: Compute hyperparameters
    m = max(20, floor(log(N)**3 / 5))
    K = max(2, floor(N / (2 * m)))
    N_exploit = N - K * m

    # Step 2: Expand — generate K diverse intermediate states
    prefixes = generate_fn(prompt, prefix=None, n=K, partial=True)

    # Step 3: Estimate — pilot sample each state
    all_completions = []
    all_scores = []
    tail_indices = []

    for prefix in prefixes:
        completions = generate_fn(prompt, prefix=prefix, n=m)
        scores = score_fn(completions)
        all_completions.extend(completions)
        all_scores.extend(scores)

        # Fit GPD to top 20% of rewards
        threshold = np.percentile(scores, 80)
        tail_data = scores[scores >= threshold] - threshold
        if len(tail_data) >= 4:
            xi, _, sigma = genpareto.fit(tail_data, floc=0)
        else:
            xi, sigma = 0.0, np.std(scores)

        tail_indices.append({
            'xi': xi, 'sigma': sigma,
            'baseline': np.mean(scores[scores >= threshold]),
            'prefix': prefix
        })

    # Step 4: Select — pick state with highest predicted potential
    def predict_value(state, n_budget):
        xi = np.clip(state['xi'], 1e-6, 0.5)  # stability clamp
        return state['baseline'] + state['sigma'] * (n_budget ** xi)

    best_state = max(tail_indices, key=lambda s: predict_value(s, N_exploit))

    # Step 5: Exploit — concentrate remaining budget on best state
    exploit_completions = generate_fn(
        prompt, prefix=best_state['prefix'], n=N_exploit
    )
    exploit_scores = score_fn(exploit_completions)
    all_completions.extend(exploit_completions)
    all_scores.extend(exploit_scores.tolist())

    # Step 6: Return global best
    best_idx = np.argmax(all_scores)
    return all_completions[best_idx], all_scores[best_idx]
```
```

## Best Practices

- **Do:** Use at least m >= 20 pilot samples per state for reliable GPD fits. Fewer samples produce unstable tail index estimates that degrade selection quality.
- **Do:** Clamp the estimated tail index xi to [0, 0.5]. Values outside this range typically indicate fitting artifacts from insufficient data rather than genuinely extreme tails.
- **Do:** Use the top 20% quantile threshold for GPD fitting (matching the reference implementation). This balances having enough tail data points against contaminating the tail with bulk distribution behavior.
- **Do:** Track and log tail indices across problems. Consistently high xi values indicate the model-reward combination benefits greatly from SLG search; near-zero xi values mean BoN is already near-optimal and SLG overhead may not be worthwhile.
- **Avoid:** Applying SLG search when total budget N < 50. The algorithm needs enough budget to support meaningful exploration (K >= 2 states, m >= 20 samples each) plus exploitation. Below this threshold, standard BoN is more practical.
- **Avoid:** Using SLG search with reward models that produce discrete/binary scores (e.g., pass/fail). GPD fitting requires continuous reward distributions with meaningful tail variation. For binary outcomes, use Bayesian methods instead.

## Error Handling

- **GPD fit failure:** If `scipy.stats.genpareto.fit` fails to converge (common with very few tail samples or degenerate reward distributions), fall back to using the empirical mean of the top quantile as V(s, N) and allocate budget proportionally to observed mean reward.
- **All tail indices near zero:** If all K states have xi < 0.01, the reward distribution is effectively light-tailed. In this case, SLG search degenerates to uniform allocation (equivalent to BoN), which is the correct behavior — there is no heavy-tail advantage to exploit.
- **Single dominant state:** If one state has much higher pilot rewards than all others regardless of tail index, the algorithm correctly selects it. The tail index only matters for distinguishing states with similar current performance but different upside potential.
- **Budget underflow:** If K * m >= N after computing hyperparameters, reduce K to max(2, floor(N / (m + 2))) to guarantee at least 2 exploitation rollouts per state. The reference implementation enforces this constraint.
- **Reward model noise:** High-variance reward models inflate tail index estimates, causing over-optimistic predictions. If you observe predicted V(s, N) consistently exceeding actual rewards by large margins, increase m to stabilize estimates or add reward score averaging.

## Limitations

- **Requires a scalar reward model.** SLG search depends on fitting continuous reward distributions. Tasks with only binary feedback (correct/incorrect) or multi-objective rewards need adaptation.
- **Single-layer assumption.** The core algorithm as described performs one round of expand-select-exploit. Multi-layer extension (nested SLG across reasoning steps) is discussed in the paper but adds significant complexity and latency.
- **Minimum budget threshold.** With fewer than ~50 total rollouts, the pilot sampling overhead (K * m) consumes too much of the budget to leave meaningful exploitation room. BoN is preferable at small scale.
- **Reward model quality dependency.** The entire approach hinges on the reward model's signal quality. If the reward model is poorly calibrated or misaligned with true task quality, tail-guided allocation amplifies errors rather than correcting them.
- **Overhead from GPD fitting.** While computationally cheap relative to LLM inference, the statistical fitting step adds latency and requires scipy. In latency-critical applications, precomputed tail indices from a calibration set can substitute for online fitting.
- **Assumes stationarity.** The tail distribution is estimated from pilot samples and assumed stable. If the reward landscape changes dramatically between exploration and exploitation (e.g., due to very different reasoning paths), predictions may be unreliable.

## Reference

**Paper:** "Predicting and improving test-time scaling laws via reward tail-guided search" by Li, Qian, Mou (2026). [arXiv:2602.01485](https://arxiv.org/abs/2602.01485v1). Look for: Algorithm 1 (SLG Search pseudocode), Theorem 3.1 (regret bound), and Section 4 (empirical validation on AIME/MATH benchmarks).

**Code:** [github.com/PotatoJnny/Scaling-Law-Guided-search](https://github.com/PotatoJnny/Scaling-Law-Guided-search) — Reference implementation with auto-tuned hyperparameters, supporting Llama and Skywork reward models.