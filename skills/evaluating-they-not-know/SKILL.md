---
name: "evaluating-they-not-know"
description: "Build statistically efficient LLM evaluation pipelines that combine direct accuracy with pairwise comparison signals as control variates. Use when the user asks to 'evaluate LLM accuracy on a benchmark', 'rank models with small sample sizes', 'reduce variance in LLM evaluation', 'build a model comparison pipeline', 'get tighter confidence intervals for model performance', or 'statistically compare reasoning models'."
---

# Evaluating LLMs via Comparative Signals: Semiparametric Estimation with Control Variates

This skill enables Claude to build evaluation pipelines that go beyond naive accuracy averaging when benchmarking LLMs. The core technique augments standard correctness labels with pairwise comparison signals -- having models judge which of two auxiliary reasoning chains is better -- then uses these comparisons as statistical control variates via an efficient influence function (EIF) estimator. This yields strictly tighter confidence intervals and more reliable model rankings, especially on small or difficult benchmarks where raw accuracy estimates are noisy.

## When to Use

- When the user needs to rank multiple LLMs on a math/reasoning benchmark with limited test instances (N < 100)
- When the user wants confidence intervals around model accuracy that are tighter than simple binomial intervals
- When building an evaluation harness that must produce stable rankings despite stochastic model outputs
- When evaluating on hard benchmarks (like AIME or GPQA) where most models score low and accuracy estimates swing wildly
- When the user asks to reduce evaluation cost by extracting more signal per test instance
- When comparing models where direct accuracy is similar and the user needs to break near-ties statistically

## Key Technique

**The problem:** Evaluating LLM accuracy on a benchmark with N problems gives you a sample mean with variance ~p(1-p)/N. On small benchmarks (AIME has 30 problems, GPQA Diamond has 198), this yields wide confidence intervals and unstable rankings. Running more samples is expensive.

**The insight:** Even when a model cannot solve a problem, it can often reliably judge which of two candidate solutions is better. These pairwise comparison signals are correlated with correctness and carry exploitable information. The paper treats them as control variates -- auxiliary random variables with known expectation that are subtracted from the estimator to reduce variance without introducing bias.

**The estimator:** For each problem X with model answer Y and ground truth G, generate auxiliary reasoning chains W1, W2 from helper models and collect the target model's preference V between them. Fit a regression function tau(X, Z) = E[correctness | comparisons, problem] using cross-fitting. The one-step estimator is: theta_hat = (1/N) * sum_i [ m_hat(X_i) + phi(Y_i, G_i) - tau_hat(X_i, Z_i) ], where m_hat is the marginal expectation integrated over auxiliary samples. This achieves the semiparametric efficiency bound -- no unbiased estimator can have lower asymptotic variance with the same data structure.

## Step-by-Step Workflow

1. **Define the evaluation target.** Specify the benchmark (problem set), the metric phi (typically exact-match correctness: phi(y,g) = 1 if y == g else 0), and the set of target models to rank.

2. **Sample the evaluation subset.** Select N problems from the benchmark. For small benchmarks use all problems; for large ones, sample N=50-100. Record each problem input X_i and ground truth G_i.

3. **Collect target model outputs.** For each problem X_i, query each target model to get answer Y_i. Score correctness phi(Y_i, G_i). This gives the naive estimator baseline.

4. **Generate auxiliary reasoning chains.** For each problem X_i, generate M+1 pairs of candidate solutions (W1_j, W2_j) from auxiliary models. Use two distinct auxiliary models for robustness (e.g., one strong closed-source, one open-source). M=10 is sufficient for the Monte Carlo integration step.

5. **Collect pairwise preferences.** For each problem X_i, present the first auxiliary pair (W1_1, W2_1) to the target model and ask it to judge which solution is better. Record V_i in {0, 1}. This is the comparison signal.

6. **Fit the outcome regression tau via cross-fitting.** Partition data into K=5 folds. For each fold k, fit tau_hat on the remaining 4 folds, where tau(X, Z) predicts correctness from the problem features and comparison signal. In small-sample regimes (N <= 30), use an LLM as the regressor via in-context learning instead of training a model.

7. **Compute the marginal regression m_hat.** For each problem X_i, average tau_hat over the remaining M auxiliary samples: m_hat(X_i) = (1/M) * sum_{j=2}^{M+1} tau_hat(W1_j, W2_j, V_j, X_i). This integrates out the auxiliary randomness.

8. **Calculate per-instance influence scores.** For each instance i in fold k: psi_i = m_hat(X_i) + phi(Y_i, G_i) - tau_hat(W1_{i,1}, W2_{i,1}, V_{i,1}, X_i).

9. **Average influence scores to get the one-step estimate.** theta_hat_onestep = (1/N) * sum_i psi_i. Compute the standard error as se = std(psi_i) / sqrt(N) and construct a 95% CI as theta_hat +/- 1.96 * se.

10. **Rank models and report.** Repeat steps 3-9 for each target model. Rank by theta_hat_onestep. Report rankings with confidence intervals. Flag pairs whose CIs overlap as statistically indistinguishable.

## Concrete Examples

**Example 1: Ranking 5 models on GPQA Diamond with N=50**

User: "I have 5 models and want to rank them on GPQA Diamond, but I can only afford 50 problems. How do I get reliable rankings?"

Approach:
1. Sample 50 problems from GPQA Diamond with ground truth answers.
2. Query all 5 target models on each problem, record correctness.
3. For each problem, generate 11 solution pairs from GPT-4o-mini and DeepSeek-V3.
4. Have each target model judge the first pair (pairwise preference).
5. Fit tau using 5-fold cross-fitting (logistic regression or LLM-as-regressor on the comparison + problem features).
6. Compute one-step estimates and CIs for each model.

Output:
```
Model Rankings (GPQA Diamond, N=50, 95% CI):
1. Model-A: 42.0% [36.1%, 47.9%]  (naive: 40.0% [26.5%, 53.5%])
2. Model-B: 38.5% [33.0%, 44.0%]  (naive: 38.0% [24.7%, 51.3%])
3. Model-C: 34.2% [28.8%, 39.6%]  (naive: 36.0% [22.9%, 49.1%])
4. Model-D: 28.7% [23.5%, 33.9%]  (naive: 30.0% [17.3%, 42.7%])
5. Model-E: 22.1% [17.2%, 27.0%]  (naive: 22.0% [10.7%, 33.3%])

Variance reduction: CIs are ~55% narrower than naive binomial intervals.
Models A and B are statistically indistinguishable at alpha=0.05.
```

**Example 2: Evaluating on AIME 2025 (N=15, extreme small-sample)**

User: "I need to compare two reasoning models on AIME 2025. Only 15 problems available."

Approach:
1. Use all 15 AIME problems. Collect correctness for both models.
2. Generate 11 auxiliary pairs per problem from two helper models.
3. Collect pairwise preferences from each target model.
4. Since N=15, skip cross-fitting. Use an LLM (e.g., Gemini Flash) as a fixed semantic regressor: prompt it with the problem, the auxiliary pair, and the preference, and ask it to predict probability of correctness.
5. Compute one-step estimates.

Output:
```
Model Comparison (AIME 2025, N=15):
  Model-X: 33.3% [22.8%, 43.8%]  (naive: 33.3% [9.5%, 57.2%])
  Model-Y: 26.7% [17.1%, 36.3%]  (naive: 26.7% [4.3%, 49.1%])

One-step CI width: ~20pp vs naive ~48pp.
Conclusion: Model-X preferred but difference not significant (p=0.31).
```

**Example 3: Python implementation skeleton**

User: "Write me code for the one-step estimator."

```python
import numpy as np
from sklearn.model_selection import KFold
from sklearn.linear_model import LogisticRegression

def one_step_estimator(
    correctness: np.ndarray,       # phi(Y_i, G_i), shape (N,)
    comparison_features: np.ndarray, # Z_i features, shape (N, M+1, d)
    n_folds: int = 5,
    n_mc_samples: int = 10
) -> dict:
    """
    Semiparametric one-step estimator with comparison control variates.

    comparison_features[:, 0, :] = first auxiliary sample (used in influence)
    comparison_features[:, 1:, :] = remaining M samples (used for m_hat)
    """
    N = len(correctness)
    psi = np.zeros(N)
    kf = KFold(n_splits=n_folds, shuffle=True, random_state=42)

    for train_idx, test_idx in kf.split(correctness):
        # Fit tau on training fold
        X_train = comparison_features[train_idx, 0, :]
        y_train = correctness[train_idx]
        reg = LogisticRegression().fit(X_train, y_train)

        # Compute tau_hat on test fold (first auxiliary sample)
        X_test_first = comparison_features[test_idx, 0, :]
        tau_hat = reg.predict_proba(X_test_first)[:, 1]

        # Compute m_hat via Monte Carlo over remaining M samples
        m_hat = np.zeros(len(test_idx))
        for j in range(1, n_mc_samples + 1):
            X_test_j = comparison_features[test_idx, j, :]
            m_hat += reg.predict_proba(X_test_j)[:, 1]
        m_hat /= n_mc_samples

        # Influence scores
        psi[test_idx] = m_hat + correctness[test_idx] - tau_hat

    theta_hat = np.mean(psi)
    se = np.std(psi, ddof=1) / np.sqrt(N)
    ci_lower = theta_hat - 1.96 * se
    ci_upper = theta_hat + 1.96 * se

    return {
        "estimate": theta_hat,
        "std_error": se,
        "ci_95": (ci_lower, ci_upper),
        "naive_estimate": np.mean(correctness),
        "naive_se": np.std(correctness, ddof=1) / np.sqrt(N),
    }
```

## Best Practices

- **Do:** Use heterogeneous auxiliary models (e.g., one closed-source, one open-source) for generating reasoning chain pairs. This increases the informativeness of comparison signals.
- **Do:** Use cross-fitting (K=5) for the nuisance regression to avoid overfitting bias. Never fit tau on the same fold you evaluate on.
- **Do:** Generate at least M=10 auxiliary samples per problem for the Monte Carlo integration of m_hat. Fewer samples add unnecessary noise to the marginal estimate.
- **Do:** Report confidence intervals alongside point estimates. The entire value of this method is in uncertainty quantification.
- **Avoid:** Using this method when you have a large benchmark (N > 500) and models are well-separated in accuracy. Naive averaging is fine there -- the overhead of generating auxiliary pairs is not worthwhile.
- **Avoid:** Using the target model itself as an auxiliary model. The auxiliary chains must come from different models to provide independent variation.
- **Avoid:** Skipping the m_hat computation and using only tau_hat. The marginal integration is essential for the efficiency bound to hold.

## Error Handling

- **tau regression fails to converge:** Fall back to LLM-as-regressor (prompt an LLM to predict correctness probability given the comparison features). This is the recommended approach for N <= 30 regardless.
- **All comparison preferences are identical:** The comparison signal carries no information (tau = m, so no variance reduction). Fall back to naive estimation and report this.
- **Auxiliary models refuse or produce empty outputs:** Retry with temperature > 0. If persistent, substitute a different auxiliary model. Each problem needs valid auxiliary pairs.
- **Negative or >1 estimates:** The one-step estimator is not bounded to [0,1]. Clip to [0,1] for reporting, but note this in results. With sufficient N the issue is rare.
- **Cross-fitting fold too small:** With N < 25, each fold may have < 5 instances. Switch to the LLM-as-fixed-regressor approach (no cross-fitting needed since the regressor is not fit on the evaluation data).

## Limitations

- Requires API access to auxiliary models and the target model's ability to perform pairwise comparisons -- not just answer questions. Models without comparison/judging capability cannot be evaluated this way.
- The variance reduction depends on comparison signals being informative about correctness. On trivial benchmarks where all models score near 100%, comparisons add little signal.
- Generating M+1 auxiliary pairs per problem multiplies API costs by roughly 2*(M+1) calls per problem beyond the base evaluation. For M=10 this is 22x more generation calls.
- The method assumes independent and identically distributed test instances. Benchmarks with correlated problems (e.g., multi-part questions) may violate this.
- Asymptotic normality requires moderate N. For N < 10 the confidence intervals from this method are not reliable.
- The technique improves estimation of a fixed model's accuracy. It does not address distribution shift, contamination, or benchmark saturation.

## Reference

Dong, Z., Zhang, Z., Zhou, Y., Jin, C., & Wu, R. (2026). *Evaluating LLMs When They Do Not Know the Answer: Statistical Evaluation of Mathematical Reasoning via Comparative Signals.* arXiv:2602.03061. Key sections: Proposition 3.1 (EIF derivation), Algorithm 1 (one-step estimator pseudocode), Theorem 4.1 (variance reduction guarantee), Table 1 (experimental results on GPQA/AIME/GSM8K).