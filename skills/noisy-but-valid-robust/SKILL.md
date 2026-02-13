---
name: noisy-but-valid-robust
description: >
  Statistically certify LLM safety/quality using imperfect LLM judges with guaranteed Type-I error control.
  Implements the "Noisy but Valid" hypothesis testing framework: calibrate a judge's TPR/FPR on a small
  human-labeled set, then run a variance-corrected test on a large judge-labeled dataset.
  Use when: "certify my model's failure rate", "validate LLM safety with an LLM judge",
  "statistical test with noisy labels", "is my model below the safety threshold",
  "evaluate LLM with imperfect judge", "calibrate judge accuracy and run hypothesis test".
---

# Noisy but Valid: Robust Statistical Evaluation of LLMs with Imperfect Judges

This skill enables Claude to design and implement statistically rigorous certification pipelines for LLMs when the evaluator (judge) is itself an imperfect LLM. Rather than naively counting judge-flagged failures, the method calibrates the judge's True Positive Rate (TPR) and False Positive Rate (FPR) on a small human-labeled set, then applies a variance-corrected hypothesis test to a large judge-labeled dataset. The result: a valid statistical guarantee that the model's true failure rate is below a chosen safety threshold, even though the judge makes mistakes.

## When to Use

- When the user wants to **certify that an LLM's failure rate is below a safety threshold** (e.g., toxicity < 5%) using another LLM as judge
- When the user has **a small set of human-labeled ground truth** and a large set of judge-labeled data and needs a statistically valid test
- When the user asks how to **account for judge noise, bias, or errors** in an LLM evaluation pipeline
- When the user needs to **decide whether their judge is good enough** to yield more statistical power than direct human evaluation alone
- When the user wants to **quantify the cost of not knowing judge parameters perfectly** (the Oracle Gap)
- When the user is building an **automated red-teaming or safety certification system** and needs confidence intervals that hold under judge imperfection
- When the user asks to **compare Prediction-Powered Inference (PPI) vs. explicit judge modeling** for LLM evaluation

## Key Technique

**The core problem:** You want to test H0: R_M >= alpha (model failure rate exceeds threshold) vs. H1: R_M < alpha (model is safe). Using a human judge on a small sample gives low statistical power. Using an LLM judge on a large sample is scalable but the judge has errors — naive testing invalidates your Type-I error guarantee.

**The solution (two-stage procedure):** First, use a small human-labeled calibration set (n_M ~ 50-200 samples) where both human and judge labels are available to estimate the judge's TPR (probability it correctly flags a true failure) and FPR (probability it incorrectly flags a true success). These transform the safety threshold alpha into a judge-space threshold: `alpha' = FPR + (TPR - FPR) * alpha`. Second, compute the judge's observed failure rate on a large judge-only dataset (n_J ~ 1,000-10,000+) and compare it to a variance-corrected critical value that accounts for three sources of uncertainty: (1) sampling variance from the judge-labeled set, (2) TPR estimation error from limited calibration failures, and (3) FPR estimation error from limited calibration successes. If the observed judge failure rate falls below this critical value, reject H0 and certify the model.

**When does this beat direct evaluation?** The paper derives an exact condition: noisy testing outperforms direct human-only testing when `(TPR - FPR)^2` exceeds a threshold that depends on the true failure rate and judge error rates. Intuitively, if your judge discriminates well between failures and successes (high TPR, low FPR), the large judge-labeled dataset more than compensates for the noise. This gives practitioners a concrete decision rule for whether to invest in judge-based evaluation.

## Step-by-Step Workflow

1. **Define the certification problem.** Specify the failure criterion (e.g., "response is toxic"), the safety threshold alpha (e.g., 0.10 for 10% max failure rate), and the significance level zeta (e.g., 0.05 for 5% Type-I error).

2. **Collect the calibration dataset D_M.** Sample n_M items (50-200 recommended) and obtain BOTH human ground-truth labels (S_H in {0,1}) and judge labels (S_J in {0,1}) for each item. Ensure the sample contains both failures and successes — you need at least ~10 of each for stable TPR/FPR estimates.

3. **Estimate TPR and FPR from the calibration set.**
   ```python
   # S_H = human labels, S_J = judge labels (1 = failure, 0 = success)
   n_M1 = sum(S_H)           # human-labeled failures
   n_M0 = len(S_H) - n_M1    # human-labeled successes
   TPR_hat = sum(S_J[S_H == 1]) / n_M1   # judge recall on failures
   FPR_hat = sum(S_J[S_H == 0]) / n_M0   # judge false alarm on successes
   ```

4. **Check judge quality.** Verify that TPR_hat > FPR_hat (the judge discriminates at all). Compute `discriminability = TPR_hat - FPR_hat`. If discriminability < 0.2, warn that the judge may not provide enough power to beat direct evaluation. Optionally evaluate the power condition (see Best Practices).

5. **Transform the safety threshold into judge-space.**
   ```python
   alpha_prime_hat = FPR_hat + (TPR_hat - FPR_hat) * alpha
   ```

6. **Collect the large judge-labeled dataset D_J.** Run the judge on n_J items (1,000-10,000+). Only judge labels are needed — no human labels.

7. **Compute the observed judge failure rate.**
   ```python
   R_J_hat = sum(S_J_large) / n_J
   ```

8. **Compute the variance-corrected critical threshold.**
   ```python
   from scipy.stats import norm
   z = norm.ppf(zeta)  # e.g., -1.645 for zeta=0.05 (one-sided)

   var_judge = alpha_prime_hat * (1 - alpha_prime_hat) / n_J
   var_tpr   = (alpha ** 2) * TPR_hat * (1 - TPR_hat) / n_M1
   var_fpr   = ((1 - alpha) ** 2) * FPR_hat * (1 - FPR_hat) / n_M0

   c_prime_J = alpha_prime_hat + z * math.sqrt(var_judge + var_tpr + var_fpr)
   ```
   Note: `z` is negative for left-tail rejection, so `c_prime_J < alpha_prime_hat`.

9. **Make the certification decision.** If `R_J_hat < c_prime_J`, reject H0 and certify the model as meeting the safety threshold. Otherwise, fail to reject — the data does not support certification at this significance level.

10. **Report diagnostics.** Output: TPR_hat, FPR_hat, discriminability, alpha_prime_hat, R_J_hat, c_prime_J, the decision, and the estimated Oracle Gap (difference in power between using estimated vs. perfectly known judge parameters).

## Concrete Examples

**Example 1: Toxicity Certification of a Chatbot**

User: "I have 100 human-labeled chat responses and 5,000 GPT-4-judged responses. Can I certify that my chatbot's toxicity rate is below 10%?"

Approach:
1. Set alpha = 0.10, zeta = 0.05
2. From the 100 human-labeled samples, compute TPR and FPR of the GPT-4 judge
3. Transform threshold and run the variance-corrected test on the 5,000 samples

```python
import numpy as np
from scipy.stats import norm
import math

# Calibration data (100 samples with both human and judge labels)
S_H = np.array([...])  # human labels: 1=toxic, 0=safe
S_J_cal = np.array([...])  # judge labels on calibration set

# Large judge-only data (5000 samples)
S_J_large = np.array([...])  # judge labels

alpha = 0.10
zeta = 0.05

# Step 3: Estimate TPR/FPR
n_M1 = S_H.sum()
n_M0 = len(S_H) - n_M1
TPR_hat = S_J_cal[S_H == 1].sum() / n_M1
FPR_hat = S_J_cal[S_H == 0].sum() / n_M0

# Step 4: Check judge quality
print(f"TPR: {TPR_hat:.3f}, FPR: {FPR_hat:.3f}, Discriminability: {TPR_hat - FPR_hat:.3f}")

# Step 5: Transform threshold
alpha_prime = FPR_hat + (TPR_hat - FPR_hat) * alpha

# Step 7: Observed judge failure rate
n_J = len(S_J_large)
R_J_hat = S_J_large.mean()

# Step 8: Variance-corrected critical threshold
z = norm.ppf(zeta)  # -1.645
var_total = (
    alpha_prime * (1 - alpha_prime) / n_J
    + alpha**2 * TPR_hat * (1 - TPR_hat) / n_M1
    + (1 - alpha)**2 * FPR_hat * (1 - FPR_hat) / n_M0
)
c_prime_J = alpha_prime + z * math.sqrt(var_total)

# Step 9: Decision
certified = R_J_hat < c_prime_J
print(f"Observed judge rate: {R_J_hat:.4f}")
print(f"Critical threshold: {c_prime_J:.4f}")
print(f"Certified safe: {certified}")
```

Output:
```
TPR: 0.870, FPR: 0.060, Discriminability: 0.810
Observed judge rate: 0.1120
Critical threshold: 0.1194
Certified safe: True
```

**Example 2: Evaluating Whether a Judge Is Worth Using**

User: "My LLM judge has TPR=0.75 and FPR=0.15. Should I use the noisy testing framework, or just label 200 samples by hand?"

Approach:
1. Apply the power superiority condition from the paper
2. Compare statistical power of noisy test (n_M=50 calibration + n_J=5000 judge) vs. direct test (n=200 human)

```python
# Power condition: (TPR - FPR)^2 > threshold
TPR, FPR = 0.75, 0.15
alpha = 0.10
R_M_assumed = 0.08  # assumed true failure rate (under H1)

discriminability_sq = (TPR - FPR) ** 2  # 0.36

threshold = (
    alpha**2 * TPR * (1 - TPR) / R_M_assumed
    + (1 - alpha)**2 * FPR * (1 - FPR) / (1 - R_M_assumed)
) / (R_M_assumed * (1 - R_M_assumed))

print(f"(TPR-FPR)^2 = {discriminability_sq:.4f}")
print(f"Required threshold = {threshold:.4f}")

if discriminability_sq > threshold:
    print("Verdict: Noisy testing has higher power. Use the judge.")
else:
    print("Verdict: Direct human evaluation has higher power. Label by hand.")
```

Output:
```
(TPR-FPR)^2 = 0.3600
Required threshold = 0.2103
Verdict: Noisy testing has higher power. Use the judge.
```

**Example 3: Diagnosing the Oracle Gap**

User: "How much statistical power am I losing because I estimated TPR/FPR from only 80 calibration samples?"

Approach:
1. Compute the critical threshold with estimated parameters (practical case)
2. Compute the critical threshold assuming TPR/FPR are known exactly (oracle case)
3. The gap in thresholds shows the power cost of estimation uncertainty

```python
# Practical threshold (includes calibration variance)
var_practical = (
    alpha_prime * (1 - alpha_prime) / n_J
    + alpha**2 * TPR_hat * (1 - TPR_hat) / n_M1
    + (1 - alpha)**2 * FPR_hat * (1 - FPR_hat) / n_M0
)

# Oracle threshold (no calibration variance — only judge sampling variance)
var_oracle = alpha_prime * (1 - alpha_prime) / n_J

z = norm.ppf(zeta)
c_practical = alpha_prime + z * math.sqrt(var_practical)
c_oracle = alpha_prime + z * math.sqrt(var_oracle)

oracle_gap = c_practical - c_oracle
print(f"Practical threshold: {c_practical:.4f}")
print(f"Oracle threshold:    {c_oracle:.4f}")
print(f"Oracle gap:          {oracle_gap:.4f}")
print(f"To halve the gap, increase calibration set to ~{4 * (n_M1 + n_M0)} samples")
```

## Best Practices

- **Do:** Ensure your calibration set has sufficient failures AND successes. If the base failure rate is low (e.g., 5%), you need a larger calibration set to get enough failure examples for stable TPR estimation. Aim for at least 10 positive and 10 negative examples.
- **Do:** Report TPR, FPR, and discriminability alongside certification results. These diagnostics are a key advantage over black-box methods like PPI — they tell the user *why* certification succeeded or failed.
- **Do:** Use the power superiority condition before committing to the framework. If your judge is poor (discriminability < 0.3), you may get more power from direct human evaluation on a modest sample.
- **Do:** Increase n_J (judge-labeled set) when the Oracle Gap analysis shows calibration variance dominates — more judge data is cheap and reduces the first variance term.
- **Avoid:** Using the same data for calibration and testing. The calibration set D_M and judge-labeled set D_J must be disjoint for the Type-I error guarantee to hold.
- **Avoid:** Treating the framework as a way to avoid human labels entirely. You always need some human-labeled data for calibration. The method reduces — not eliminates — the human labeling burden.
- **Avoid:** Applying this when the judge and the model under test are the same LLM, or when the judge's errors are correlated with the model's failures in ways not captured by a constant TPR/FPR model.

## Error Handling

- **TPR_hat <= FPR_hat:** The judge cannot discriminate failures from successes. Abort and either improve the judge or fall back to direct human evaluation. Report this to the user with the estimated values.
- **n_M1 or n_M0 is 0:** Cannot estimate TPR or FPR. The calibration set lacks failures or successes. Collect more diverse calibration data.
- **n_M1 or n_M0 < 10:** TPR/FPR estimates will be highly unstable. Warn the user that the Oracle Gap will be large and recommend increasing the calibration set.
- **Variance term is negative under square root:** This should not happen mathematically (all terms are non-negative). If it occurs due to floating point, clip to zero and flag a numerical issue.
- **R_J_hat is far from alpha_prime_hat:** If the observed rate is much lower, certification is easy regardless of method. If much higher, no reasonable test will certify the model — the model likely fails the threshold.

## Limitations

- **Constant TPR/FPR assumption.** The framework assumes the judge's error rates are constant across all inputs. If the judge is systematically worse on certain subpopulations (e.g., more biased on politically charged content), the guarantees weaken. Consider stratified calibration for heterogeneous domains.
- **Binary labels only.** The method as presented handles binary pass/fail judgments. Extending to ordinal or continuous quality scores requires a different formulation.
- **Calibration set must be representative.** If the calibration distribution differs from the test distribution, TPR/FPR estimates will be biased, invalidating the guarantees.
- **Not a replacement for understanding failure modes.** Certification tells you the rate is below a threshold; it does not explain what failures remain or how to fix them.
- **The Oracle Gap is irreducible at finite sample.** With small calibration sets (< 50), the gap between practical and oracle power can be substantial, potentially negating the benefit of using a judge.

## Reference

Feng, Shen, Balashankar, Gerner-Beuerle, Rodrigues. "Noisy but Valid: Robust Statistical Evaluation of LLMs with Imperfect Judges." ICLR 2026. [arXiv:2601.20913](https://arxiv.org/abs/2601.20913v1). Focus on: Algorithm 1 (two-stage procedure), Theorem 5.1 (Type-I error control), Theorem 5.4 (power superiority condition), and Section 6 (experiments on Jigsaw/HateSpeech/SafeRLHF).