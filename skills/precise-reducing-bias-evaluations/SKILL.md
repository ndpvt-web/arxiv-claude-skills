---
name: "precise-reducing-bias-evaluations"
description: "Implement the PRECISE framework to debias LLM-as-judge evaluations of search, ranking, and RAG systems by combining a small human-annotated gold set with large-scale LLM judgments using Prediction-Powered Inference. Triggers: 'evaluate search quality with LLM judges', 'debias LLM relevance judgments', 'estimate Precision@K with fewer annotations', 'combine human and LLM labels for ranking evaluation', 'reduce annotation cost for retrieval evaluation', 'correct LLM judge bias in ranking metrics'"
---

# PRECISE: Prediction-Powered Ranking Estimation for Debiased LLM Evaluation

This skill enables Claude to implement the PRECISE statistical framework for evaluating search, ranking, and RAG systems. Instead of relying solely on expensive human annotations or biased LLM judgments, PRECISE combines ~100 human-annotated queries with ~10,000 LLM-judged queries using Prediction-Powered Inference (PPI) to produce unbiased metric estimates with valid confidence intervals. The technique corrects systematic LLM-judge bias while reducing annotation costs by 95%.

## When to Use

- When the user needs to evaluate a search or retrieval system but cannot afford to annotate thousands of query-result pairs by hand
- When the user has LLM-generated relevance labels (from GPT-4, Claude, etc.) and wants to correct for their systematic bias
- When computing Precision@K, nDCG, or other ranking metrics and the user wants confidence intervals, not just point estimates
- When the user is running an A/B test on a query reformulation, reranking, or RAG pipeline and needs reliable uplift estimates from minimal human labels
- When the user asks how many human annotations are needed to trust LLM-based evaluation
- When building an evaluation pipeline that mixes human and automated relevance judgments

## Key Technique

**Prediction-Powered Inference (PPI)** is a statistical framework that treats LLM predictions as a noisy proxy for human labels and uses a small "gold" set of human annotations to estimate and correct the bias. The PRECISE estimator is:

```
mu_PPI = lambda * (1/N) * sum(phi_hat_unlabeled) + (1/n) * sum(phi_gold - lambda * phi_hat_gold)
```

Here, `phi_hat` is the metric computed from LLM labels, `phi` is the metric from human labels, `n` is the small gold set (~100 queries), `N` is the large unlabeled set (~10,000 queries), and `lambda` in [0,1] is tuned to minimize variance. The second term acts as a **rectifier** -- it measures the gap between LLM and human judgments on the gold set, then subtracts that bias from the full LLM-based estimate. The estimator is unbiased for any lambda > 0.

**Sub-instance extension**: Standard PPI operates on instance-level labels. PRECISE extends it to query-document pairs. For Precision@K, each query has K retrieved documents, each with a binary relevance label. The metric is `phi(y_hat, y) = y_hat^T y / K` (dot product of predicted and true K-hot relevance vectors, divided by K). The key complexity reduction reformulates the integration space from O(2^|C|) -- all possible relevance assignments over the full corpus -- to O(2^K) by collapsing all non-retrieved documents into a single zero-probability mass. Since K is typically 5-10, this makes the computation tractable.

**Calibration**: Raw LLM confidence scores are poorly calibrated. PRECISE uses isotonic regression on the gold set to map LLM-stated confidence levels (e.g., "Almost Certain", "Likely", "About Even") to calibrated probabilities, then constructs per-document relevance probabilities as `p(y) = prod_{k=1}^{K} p'(d_k)^{y_k} * (1 - p'(d_k))^{1 - y_k}`.

## Step-by-Step Workflow

1. **Collect LLM relevance judgments on the full dataset.** For each query in your evaluation set (N ~ 10,000), prompt an LLM to judge the relevance of each of the top-K retrieved documents. Extract both a binary label (relevant/irrelevant) and a confidence level. Store as `(query_id, doc_id, llm_label, llm_confidence)` tuples.

2. **Sample and annotate a gold set.** Randomly sample n ~ 100 queries from the full set. Have human annotators label the same top-K documents for each sampled query with binary relevance. Store as `(query_id, doc_id, human_label)` tuples. Ensure the gold set is a proper subset of the LLM-judged set.

3. **Calibrate LLM confidence scores.** Using the gold set where both human and LLM labels exist, fit an isotonic regression model mapping LLM confidence to empirical relevance probability. Apply this calibration to all LLM judgments to get calibrated per-document probabilities `p'(d_k)`.

4. **Compute per-query metrics on the gold set.** For each gold query, compute Precision@K using human labels: `phi_gold_i = sum(human_relevant_in_top_k) / K`. Also compute the LLM-based metric on the same queries: `phi_hat_gold_i` using calibrated LLM labels.

5. **Compute per-query metrics on the unlabeled set.** For each non-gold query, compute Precision@K from the calibrated LLM labels: `phi_hat_unlabeled_j`.

6. **Tune lambda to minimize variance.** Compute `lambda_opt = Cov(phi_gold - phi_hat_gold, phi_hat_unlabeled) / Var(phi_hat_unlabeled)`, clipped to [0, 1]. This balances reliance on the LLM predictions against the bias correction term.

7. **Compute the PRECISE estimate.** Apply the PPI formula:
   ```python
   rectifier = np.mean(phi_gold - lambda_opt * phi_hat_gold)
   mu_precise = lambda_opt * np.mean(phi_hat_unlabeled) + rectifier
   ```

8. **Construct confidence intervals.** Estimate the variance of `mu_precise` as:
   ```python
   var_estimate = (lambda_opt**2 * np.var(phi_hat_unlabeled) / N) + (np.var(phi_gold - lambda_opt * phi_hat_gold) / n)
   ci = (mu_precise - 1.96 * sqrt(var_estimate), mu_precise + 1.96 * sqrt(var_estimate))
   ```

9. **Compare treatments (for A/B evaluation).** To estimate uplift between a treatment and control, compute `mu_precise` for each arm independently and take the difference. Propagate confidence intervals via `var_diff = var_treatment + var_control`.

10. **Validate and report.** Check that the confidence interval has the expected coverage by running on held-out gold data if available. Report the point estimate, 95% CI, the estimated LLM bias (mean of `phi_hat_gold - phi_gold`), and the effective sample size gain.

## Concrete Examples

**Example 1: Evaluating a search reranker with debiased Precision@5**

User: "I have 8,000 queries with GPT-4 relevance labels on the top-5 results, plus 120 queries with human labels. How do I get an unbiased Precision@5 estimate?"

Approach:
1. Load both datasets and align on query IDs
2. Compute per-query Precision@5 from human labels on the 120 gold queries
3. Compute per-query Precision@5 from GPT-4 labels on all 8,000 queries
4. Apply the PRECISE estimator with lambda tuning

```python
import numpy as np
from sklearn.isotonic import IsotonicRegression

# Step 1: Load data
# gold_metrics[i] = Precision@5 from human labels for gold query i
# llm_metrics_gold[i] = Precision@5 from LLM labels for gold query i
# llm_metrics_unlabeled[j] = Precision@5 from LLM labels for unlabeled query j

gold_metrics = np.array([sum(human_labels[q][:5]) / 5 for q in gold_queries])
llm_metrics_gold = np.array([sum(llm_labels[q][:5]) / 5 for q in gold_queries])
llm_metrics_unlabeled = np.array([sum(llm_labels[q][:5]) / 5 for q in unlabeled_queries])

n = len(gold_metrics)        # ~120
N = len(llm_metrics_unlabeled)  # ~7880

# Step 2: Tune lambda
cov = np.cov(gold_metrics - llm_metrics_gold, llm_metrics_gold)[0, 1]
lambda_opt = np.clip(cov / np.var(llm_metrics_gold), 0, 1)

# Step 3: Compute PRECISE estimate
rectifier = np.mean(gold_metrics - lambda_opt * llm_metrics_gold)
mu_precise = lambda_opt * np.mean(llm_metrics_unlabeled) + rectifier

# Step 4: Confidence interval
var_est = (lambda_opt**2 * np.var(llm_metrics_unlabeled) / N +
           np.var(gold_metrics - lambda_opt * llm_metrics_gold) / n)
ci_lower = mu_precise - 1.96 * np.sqrt(var_est)
ci_upper = mu_precise + 1.96 * np.sqrt(var_est)

print(f"Precision@5 = {mu_precise:.4f} [{ci_lower:.4f}, {ci_upper:.4f}]")
print(f"LLM bias (gold set): {np.mean(llm_metrics_gold - gold_metrics):.4f}")
print(f"Lambda: {lambda_opt:.3f}")
```

Output:
```
Precision@5 = 0.7234 [0.6891, 0.7577]
LLM bias (gold set): +0.0412
Lambda: 0.847
```

**Example 2: A/B test uplift for a query reformulation system**

User: "I'm testing two query reformulation strategies. I have LLM relevance labels for 10k queries per arm and 100 human-annotated queries per arm. How do I estimate which reformulation is better?"

Approach:
1. Compute `mu_precise` independently for each treatment arm
2. Compute the difference and propagate confidence intervals
3. Report whether the CI excludes zero (statistically significant)

```python
def precise_estimate(gold_metrics, llm_gold, llm_unlabeled):
    n, N = len(gold_metrics), len(llm_unlabeled)
    cov = np.cov(gold_metrics - llm_gold, llm_gold)[0, 1]
    lam = np.clip(cov / (np.var(llm_gold) + 1e-10), 0, 1)
    rect = np.mean(gold_metrics - lam * llm_gold)
    mu = lam * np.mean(llm_unlabeled) + rect
    var = (lam**2 * np.var(llm_unlabeled) / N +
           np.var(gold_metrics - lam * llm_gold) / n)
    return mu, var

mu_t1, var_t1 = precise_estimate(gold_t1, llm_gold_t1, llm_unlabeled_t1)
mu_t2, var_t2 = precise_estimate(gold_t2, llm_gold_t2, llm_unlabeled_t2)

uplift = mu_t2 - mu_t1
uplift_se = np.sqrt(var_t1 + var_t2)
ci = (uplift - 1.96 * uplift_se, uplift + 1.96 * uplift_se)

print(f"Uplift (T2 - T1): {uplift:+.4f} [{ci[0]:+.4f}, {ci[1]:+.4f}]")
significant = ci[0] > 0 or ci[1] < 0
print(f"Statistically significant: {significant}")
```

Output:
```
Uplift (T2 - T1): +0.0321 [+0.0058, +0.0584]
Statistically significant: True
```

**Example 3: Calibrating LLM confidence before applying PRECISE**

User: "My LLM judge outputs confidence levels like 'Highly Relevant', 'Somewhat Relevant', 'Irrelevant'. How do I calibrate these?"

Approach:
1. Map verbal confidence to numeric scores
2. Fit isotonic regression on gold set
3. Use calibrated probabilities for PRECISE

```python
from sklearn.isotonic import IsotonicRegression

# Map verbal to numeric
confidence_map = {"Irrelevant": 0.1, "Somewhat Relevant": 0.5,
                  "Relevant": 0.75, "Highly Relevant": 0.95}
llm_raw_scores = np.array([confidence_map[c] for c in llm_confidences_gold])
human_binary = np.array(human_labels_gold)  # 0 or 1

# Fit isotonic regression (monotonic calibration)
iso = IsotonicRegression(out_of_bounds="clip")
iso.fit(llm_raw_scores, human_binary)

# Apply calibration to all LLM scores
calibrated_all = iso.predict(np.array([confidence_map[c] for c in llm_confidences_all]))

# Now use calibrated scores to compute per-document P(relevant)
# and derive Precision@K estimates for PRECISE
```

## Best Practices

- **Do** randomly sample the gold set from the same distribution as the full query set. Stratified sampling by query type or frequency improves representativeness.
- **Do** calibrate LLM confidence scores with isotonic regression before computing metrics. Raw LLM probabilities are systematically miscalibrated.
- **Do** report both the PRECISE point estimate and confidence interval. The CI width tells you whether your gold set is large enough.
- **Do** check the sign and magnitude of the estimated LLM bias on the gold set. A large bias (>5% absolute) indicates the LLM judge needs prompt tuning or the gold set is critical for correction.
- **Avoid** using fewer than 50 gold queries. Below this threshold, the rectifier variance dominates and PRECISE offers no advantage over gold-only estimation.
- **Avoid** assuming the LLM bias is constant across query segments. Compute bias separately for head vs. tail queries, or by query category, if your gold set is large enough (30+ per segment).
- **Avoid** increasing the unlabeled set beyond 100x the gold set size without checking for diminishing returns. Going from 100x to 2000x unlabeled data yields minimal variance reduction.

## Error Handling

- **Gold set too small**: If n < 50, the rectifier variance is high. Fall back to reporting LLM-only metrics with a warning about potential bias, or invest in more annotations.
- **Lambda collapses to 0**: This means LLM predictions have no useful signal beyond what the gold set provides. Check that the LLM judge prompt is reasonable and that calibration was applied. A lambda of 0 reduces PRECISE to a gold-only estimator.
- **Lambda goes to 1**: The LLM is well-aligned with humans. The estimate is essentially the full LLM average with a small bias correction. This is the best case.
- **Confidence interval wider than gold-only**: This can happen when `lambda` is poorly estimated with very small gold sets. Increase gold set size or set `lambda = 0` to fall back gracefully.
- **Non-overlapping gold and LLM sets**: The gold queries must be a subset of the LLM-judged queries. If they are disjoint, you cannot compute the rectifier. Ensure LLM labels exist for all gold queries.

## Limitations

- PRECISE assumes the gold set is drawn i.i.d. from the same distribution as the unlabeled set. If human annotators are sourced differently from the LLM's training distribution (e.g., domain experts vs. crowd workers), the bias correction may not transfer.
- The framework is designed for metrics decomposable at the query level (Precision@K, per-query nDCG). It does not directly apply to list-wise metrics that depend on the joint distribution across queries.
- The O(2^K) computation is tractable for K <= 15 but becomes expensive for large K. For Precision@50 or recall-oriented metrics over large result lists, approximate methods are needed.
- The method corrects for systematic (average) bias but not for instance-conditional bias. If the LLM is biased differently on easy vs. hard queries, a single rectifier may under-correct on some segments.
- Binary relevance is assumed. Extending to graded relevance (for nDCG with multi-level labels) requires adapting the integration space, which increases complexity.

## Reference

[PRECISE: Reducing the Bias of LLM Evaluations Using Prediction-Powered Ranking Estimation](https://arxiv.org/abs/2601.18777v1) (Divekar & Majumder, AAAI 2026 IAAI). Key sections: Section 3 for the PPI extension to sub-instance annotations, Section 4 for the metric-space complexity reduction from O(2^|C|) to O(2^K), and Section 5 for experimental validation on e-commerce search with calibrated Claude 3 Sonnet as the LLM judge.