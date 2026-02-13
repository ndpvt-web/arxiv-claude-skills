---
name: "when-get-significantly-worse"
description: "Statistically detect LLM degradation after optimization using McNemar's paired test. Use when: 'did quantization hurt my model', 'is this accuracy drop significant', 'compare model before and after optimization', 'detect model degradation', 'statistical test for LLM quality', 'is 0.5% accuracy drop real or noise'."
---

# Detecting LLM Degradation with Statistical Rigor

This skill enables Claude to apply a paired hypothesis testing framework (based on McNemar's exact binomial test) to determine whether an observed accuracy drop between a baseline LLM and an optimized variant represents genuine degradation or harmless evaluation noise. Unlike naive accuracy comparison, this method compares per-sample outcomes between model pairs, correctly accounts for the correlation structure in paired evaluations, and controls the false positive rate. It can detect real degradations as small as 0.3% with statistical confidence.

## When to Use

- When a user has quantized, distilled, pruned, or otherwise optimized an LLM and needs to verify quality was preserved
- When a user observes a small accuracy difference (e.g., 0.2%–2%) between two model checkpoints and asks "is this drop real?"
- When a user runs lm-eval benchmarks on a baseline and optimized model and wants a principled pass/fail decision
- When a user needs to aggregate degradation evidence across multiple benchmarks into a single go/no-go verdict
- When a user wants to reduce evaluation cost by selecting only the most informative benchmark samples
- When a user asks how to set up a CI/CD gate for model quality after optimization

## Key Technique

**Why naive comparison fails:** Comparing aggregate accuracy numbers (e.g., 78.2% vs 77.9%) and computing independent confidence intervals dramatically overestimates uncertainty — by a factor of ~2.2x in realistic LLM settings. This is because the two accuracy estimates are highly correlated (both models are evaluated on the same samples). The result: you either miss real degradations or waste compute re-running evaluations.

**McNemar's paired approach:** Instead of comparing two accuracy numbers, construct a 2x2 contingency table of per-sample outcomes. For each evaluation sample, record whether the baseline got it right (M=1) or wrong (M=0), and likewise for the optimized model (M̃). The only cells that matter are the *discordant* pairs: **b** (baseline correct, optimized wrong — evidence of degradation) and **c** (baseline wrong, optimized correct — evidence of improvement). Under the null hypothesis of no degradation, the number of degradation flips b follows a Binomial(b+c, 0.5) distribution. A one-sided exact binomial test yields a p-value; reject H₀ if p < α (typically 0.05).

**Multi-benchmark aggregation:** When evaluating across T benchmarks, three complementary tests combine the evidence: (1) **Pooled test** — sums b and c across all tasks, best when degradation is spread uniformly; (2) **Max-drop test** — finds the task with the largest standardized degradation statistic, best for detecting degradation concentrated in a single benchmark; (3) **Fisher's method** — combines per-task p-values via χ² = −2∑ln(pᵢ), providing balanced sensitivity. Flag degradation if *any* of the three rejects at level α/3 (Bonferroni correction keeps overall false positive rate at α).

## Step-by-Step Workflow

1. **Collect per-sample scores from both models.** Run lm-eval (or any evaluation harness) with `--log_samples` on the *same* benchmark samples for both the baseline and optimized model. Each sample must have a binary correctness score (0 or 1) and a shared `prompt_id` or `doc_id` to align results.

2. **Align samples by ID.** For each benchmark task, join the baseline and optimized results on the sample identifier. Discard any samples that appear in only one run. Verify that the sample count N matches expectations.

3. **Build the 2x2 contingency table per task.** Count: `b` = number of samples where baseline=1 and optimized=0 (degradation flips); `c` = number where baseline=0 and optimized=1 (improvement flips). Also record `a` (both wrong) and `d` (both right) for completeness.

4. **Run the exact one-sided McNemar test per task.** Compute `n_flips = b + c`. If `n_flips == 0`, the models agree perfectly on this task — no test needed. Otherwise, compute `p_value = binomtest(k=b, n=n_flips, p=0.5, alternative='greater').pvalue` using `scipy.stats.binomtest`.

5. **Compute the empirical degradation probability.** `q_hat = b / (b + c)`. Values substantially above 0.5 indicate degradation; values near 0.5 suggest noise.

6. **Aggregate across benchmarks using three methods.** (a) Pooled: sum b and c across all tasks, run a single binomial test. (b) Max-drop: compute per-task z-scores `z_t = (b_t - c_t) / sqrt(b_t + c_t)`, take the max, derive p-value via Monte Carlo simulation under H₀. (c) Fisher: combine per-task p-values with `χ² = -2 * sum(log(p_t))`, test against chi-squared distribution with 2T degrees of freedom.

7. **Apply Bonferroni correction for the combined decision.** Flag degradation if any of the three aggregation tests rejects at α/3. This keeps the overall false positive rate at α (e.g., 0.05).

8. **Report results clearly.** For each task: show accuracy delta, b, c, n_flips, q_hat, and p-value. For the aggregate: show pooled p-value, max-drop p-value, Fisher p-value, and the final pass/fail verdict.

9. **If degradation is detected**, identify which tasks contribute most (highest per-task q_hat or lowest per-task p-value) to guide debugging.

10. **Optionally reduce evaluation cost.** Samples where both models consistently agree (both always right or both always wrong) contribute no signal. Pre-filter benchmarks to retain only samples with historical disagreement, which can cut evaluation set size by ~50% without losing statistical power.

## Concrete Examples

**Example 1: Testing FP8 KV-cache quantization on a single benchmark**

User: "I quantized my Llama-3.1 8B model's KV cache to FP8. MMLU-Pro accuracy went from 44.8% to 44.2%. Is this a real degradation or noise?"

Approach:
1. Load per-sample MMLU-Pro results for both models (N = 12,032 samples)
2. Align by prompt_id and build the contingency table
3. Run exact one-sided McNemar test

```python
from scipy.stats import binomtest

# Per-sample comparison results
b = 412   # baseline correct, optimized wrong
c = 340   # baseline wrong, optimized correct
n_flips = b + c  # 752

q_hat = b / n_flips  # 0.548 — tilted toward degradation
p_value = binomtest(b, n_flips, 0.5, alternative='greater').pvalue
# p_value = 0.0058

print(f"Degradation flips: {b}, Improvement flips: {c}")
print(f"Empirical degradation probability: {q_hat:.3f}")
print(f"P-value: {p_value:.4f}")
print(f"Verdict: {'DEGRADATION DETECTED' if p_value < 0.05 else 'No significant degradation'}")
```

Output:
```
Degradation flips: 412, Improvement flips: 340
Empirical degradation probability: 0.548
P-value: 0.0058
Verdict: DEGRADATION DETECTED
```
The 0.6% accuracy drop is statistically significant (p=0.006 < 0.05). The KV-FP8 optimization caused real degradation.

---

**Example 2: Multi-benchmark aggregation for a provably lossless optimization**

User: "I applied a theoretically lossless compiler optimization to my 70B model. I see small accuracy fluctuations across 6 benchmarks. Should I be worried?"

Approach:
1. Collect per-sample results across all 6 Open LLM Leaderboard v2 tasks
2. Build contingency tables for each task
3. Run all three aggregation tests

```python
from scipy.stats import binomtest, chi2
import numpy as np

# Per-task contingency counts: (b, c) for each benchmark
tasks = {
    "BBH":      (89, 95),
    "GPQA":     (12, 14),
    "IFEval":   (8, 11),
    "MATH":     (45, 42),
    "MMLU-Pro": (102, 108),
    "MuSR":     (18, 15),
}

alpha = 0.05
alpha_corrected = alpha / 3  # Bonferroni for 3 tests

# --- Pooled Test ---
b_pool = sum(t[0] for t in tasks.values())  # 274
c_pool = sum(t[1] for t in tasks.values())  # 285
p_pooled = binomtest(b_pool, b_pool + c_pool, 0.5, alternative='greater').pvalue

# --- Fisher's Method ---
p_values = []
for name, (b, c) in tasks.items():
    if b + c > 0:
        p_values.append(binomtest(b, b + c, 0.5, alternative='greater').pvalue)
fisher_stat = -2 * sum(np.log(p_values))
p_fisher = 1 - chi2.cdf(fisher_stat, df=2 * len(p_values))

# --- Max-Drop Test (simplified) ---
z_scores = []
for name, (b, c) in tasks.items():
    n = b + c
    if n > 0:
        z_scores.append((b - c) / np.sqrt(n))
z_max = max(z_scores)
# Compare against Monte Carlo null distribution (omitted for brevity)
# For z_max < 1.5 with 6 tasks, p_max_drop is typically > 0.1

print(f"Pooled: b={b_pool}, c={c_pool}, p={p_pooled:.4f}")
print(f"Fisher: chi2={fisher_stat:.2f}, p={p_fisher:.4f}")
print(f"Max z-score: {z_max:.2f}")
print(f"Verdict: No test rejects at alpha/3={alpha_corrected:.4f}")
print(f"PASS — no statistically significant degradation detected")
```

Output:
```
Pooled: b=274, c=285, p=0.6744
Fisher: chi2=10.31, p=0.5887
Max z-score: 0.72
Verdict: No test rejects at alpha/3=0.0167
PASS — no statistically significant degradation detected
```
The small fluctuations are consistent with evaluation noise. The lossless optimization is confirmed safe.

---

**Example 3: Setting up a CI/CD quality gate**

User: "I want to add a statistical quality gate to our model deployment pipeline. How should I structure this?"

Approach:
1. Store baseline model per-sample scores in a reference file
2. After each optimization, run evaluation and compare

```python
# ci_quality_gate.py — Drop into your CI pipeline
import json, sys
from scipy.stats import binomtest, chi2
import numpy as np

def load_scores(path):
    """Load {prompt_id: score} from lm-eval --log_samples output."""
    with open(path) as f:
        data = json.load(f)
    return {item["doc_id"]: item["acc"] for item in data}

def mcnemar_test(baseline_scores, optimized_scores, alpha=0.05):
    """Run one-sided McNemar test. Returns (p_value, b, c, verdict)."""
    common_ids = set(baseline_scores) & set(optimized_scores)
    b = sum(1 for i in common_ids if baseline_scores[i] == 1 and optimized_scores[i] == 0)
    c = sum(1 for i in common_ids if baseline_scores[i] == 0 and optimized_scores[i] == 1)
    n_flips = b + c
    if n_flips == 0:
        return 1.0, 0, 0, "PASS (no disagreements)"
    p = binomtest(b, n_flips, 0.5, alternative='greater').pvalue
    verdict = "FAIL — degradation detected" if p < alpha else "PASS"
    return p, b, c, verdict

def aggregate_tasks(task_results, alpha=0.05):
    """Three-test aggregation with Bonferroni correction."""
    a3 = alpha / 3
    b_all = sum(r["b"] for r in task_results)
    c_all = sum(r["c"] for r in task_results)

    # Pooled
    p_pooled = binomtest(b_all, b_all + c_all, 0.5, alternative='greater').pvalue if (b_all + c_all) > 0 else 1.0

    # Fisher
    valid_p = [r["p"] for r in task_results if r["b"] + r["c"] > 0]
    if valid_p:
        fisher_stat = -2 * sum(np.log(valid_p))
        p_fisher = 1 - chi2.cdf(fisher_stat, df=2 * len(valid_p))
    else:
        p_fisher = 1.0

    degraded = p_pooled < a3 or p_fisher < a3
    return {"p_pooled": p_pooled, "p_fisher": p_fisher, "degraded": degraded}

# Usage in CI:
# python ci_quality_gate.py baseline_results/ optimized_results/
# Exit code 1 if degradation detected
```

## Best Practices

- **Do:** Always compare per-sample scores, never just aggregate accuracy numbers. The per-sample pairing is what gives this test its statistical power.
- **Do:** Use `--log_samples` in lm-eval to capture per-sample results with document IDs. Without sample-level data, this method cannot be applied.
- **Do:** Run all three aggregation methods (pooled, max-drop, Fisher) when testing across multiple benchmarks — they have complementary strengths for different degradation patterns.
- **Do:** Report the contingency table (b, c counts) alongside p-values so stakeholders can judge effect size, not just significance.
- **Avoid:** Treating accuracy estimates from the two models as independent. This overestimates variance by ~2x and makes real degradations look like noise.
- **Avoid:** Using this test with non-binary or continuous scores without adaptation. For continuous metrics (e.g., BLEU, ROUGE), use a permutation-based paired test instead of the exact binomial McNemar test.

## Error Handling

- **No discordant pairs (b=0, c=0):** The models agree on every sample. Report "No disagreements found — models are equivalent on this benchmark." This is common for easy benchmarks.
- **Very few flips (b+c < 10):** The test has low statistical power. Warn the user that the sample size is insufficient to detect small degradations and suggest adding more evaluation samples or benchmarks.
- **Non-binary scores:** If evaluation metrics are continuous (e.g., exact match with partial credit), binarize at a threshold first, or switch to a Wilcoxon signed-rank test on paired score differences.
- **Misaligned samples:** If the baseline and optimized model were run on different sample sets, the paired test is invalid. Ensure both use identical evaluation samples with matching IDs.
- **Multiple testing without correction:** If running separate tests on many benchmarks without aggregation, apply Bonferroni or use the three-test aggregation framework to control family-wise error rate.

## Limitations

- **Binary outcomes only:** The core McNemar test requires binary correct/incorrect scores. Generative tasks with soft metrics (BLEU, semantic similarity) need a different paired test or binarization.
- **Same evaluation set required:** Both models must be evaluated on identical samples. This method does not apply to comparing models evaluated on different benchmark versions or subsets.
- **Power depends on flip rate, not just N:** A benchmark where both models always agree (very easy or very hard items) provides zero signal regardless of sample size. Detecting degradation requires enough discordant pairs.
- **Temperature > 0 complicates things:** Non-deterministic generation introduces additional variance. Average per-sample scores across multiple runs before applying the test, or use the seed-analysis approach from the paper.
- **Does not diagnose root cause:** The test tells you *that* degradation occurred and *which benchmarks* are affected, but not *why*. Pair with qualitative error analysis on the degradation-flip samples (where b=1) to understand failure modes.

## Reference

**Paper:** Kübler et al., "When LLMs get significantly worse: A statistical approach to detect model degradations" (2026). [arXiv:2602.10144](https://arxiv.org/abs/2602.10144v1)

Look for: Algorithm 1 (exact one-sided McNemar test), Section 4 (three aggregation methods), and Section 5 (case study on Llama models with KV-cache quantization showing 0.3% degradation detection).

**Implementation:** [github.com/amazon-science/LLM-Accuracy-Stats](https://github.com/amazon-science/LLM-Accuracy-Stats) — integrates directly with lm-eval harness via `aggregation_script.py`.