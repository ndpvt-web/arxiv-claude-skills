---
name: "llm-assisted-logic-rule-learning"
description: "Build deterministic, interpretable anomaly detection rule sets for time series data using LLM-driven labeling, symbolic rule generation, and iterative optimization. Use when: 'detect anomalies in time series', 'build rule-based anomaly detection', 'generate logic rules from data', 'convert expert knowledge to detection rules', 'supply chain anomaly monitoring', 'create interpretable anomaly classifier'."
---

# LLM-Assisted Logic Rule Learning for Time Series Anomaly Detection

This skill enables Claude to build production-grade, deterministic anomaly detection systems for time series data by encoding domain expertise into interpretable symbolic rules. Rather than deploying an LLM at inference time (expensive, non-deterministic) or relying on unsupervised methods (misaligned with business context), this three-stage framework uses the LLM as a **compiler**: it translates human expertise into executable Python rule functions that run in milliseconds with consistent results. The technique originates from Zhang & Jain (2026), achieving 92% F1 with 4,283x speedup over direct LLM inference on supply chain data.

## When to Use

- When the user needs anomaly detection on time series data (inventory levels, server metrics, financial signals, IoT sensor streams) and wants interpretable, auditable rules rather than black-box models
- When the user asks to "build a rule-based anomaly detector" or "create detection rules from examples"
- When the user has domain knowledge about what constitutes an anomaly but needs to scale detection across thousands or millions of time series
- When unsupervised methods (Isolation Forest, autoencoders) produce too many false positives that don't match business definitions of anomalies
- When the user wants to replace expensive per-request LLM anomaly classification with deterministic rule execution
- When the user needs anomaly categories (not just binary labels) mapped to actionable business responses
- When building DevOps alerting pipelines, supply chain monitors, or security event detectors that require low-latency, explainable decisions

## Key Technique

The framework operates in three stages that separate the LLM's role from the production detection path:

**Stage 1 - LLM-as-Labeler:** Instead of hand-labeling thousands of time series windows, use the LLM as a scalable labeling operator. Present each time series sample (as numerical features or visualized plots) alongside domain-specific instructions that define what counts as anomalous in context. Use multi-model consensus (multiple LLMs vote independently) and multi-trial internal voting to produce high-confidence labels. Human experts review samples and refine prompts iteratively -- the LLM labels at scale, humans manage quality.

**Stage 2 - Symbolic Rule Generation and Optimization:** Extract qualitative reasoning from the LLM's labeling explanations ("significantly higher than historical average," "breaking established trend") and combine with computed statistical features (z-scores, rolling means, ratios, zero-rates). The LLM generates initial rule prototypes as executable Python functions with IF-THEN logic, threshold comparisons, and AND/OR compositions. Then iterate: evaluate each rule's precision/recall/F1 on labeled data, feed the confusion matrix and behavioral analysis (over-conservative? over-aggressive?) back to the LLM, and request targeted modifications. Track the optimization trajectory to prevent cyclic edits. Run multiple independent starts and select the best rule set on a held-out test split.

**Stage 3 - Rule Augmentation with Business Categories:** Once binary detection rules are finalized, use the LLM to analyze each rule's detection principle and map it to a taxonomy of business-relevant anomaly categories (e.g., "Critical Stock-Out Crisis," "Moderate Pressure," "Escalating Emergency"). This transforms raw alerts into actionable categories that drive different remediation workflows.

## Step-by-Step Workflow

1. **Define the anomaly domain context.** Write a clear specification of what constitutes an anomaly for this use case. Include: the metric being monitored, the business impact of anomalies, examples of true positives vs. acceptable variation, and any domain-specific directionality (e.g., "a sharp drop in out-of-stock ratio is good, not anomalous"). Store this as a `domain_context.md` file.

2. **Engineer statistical features from raw time series.** Compute features in deterministic Python code (not via LLM): current value, rolling mean and standard deviation (e.g., 4-week window), z-score relative to history, ratio of current value to recent values, trend slope, zero-rate, min/max over window, and first-order differences. Output a structured feature dict per time series sample.

   ```python
   def compute_features(series: list[float]) -> dict:
       recent = series[-4:]
       historical = series[:-1]
       return {
           "current_value": series[-1],
           "recent_mean": np.mean(recent),
           "recent_std": np.std(recent),
           "historical_mean": np.mean(historical),
           "z_score": (series[-1] - np.mean(historical)) / max(np.std(historical), 1e-6),
           "ratio_to_prev": series[-1] / max(series[-2], 1e-6),
           "zero_rate": sum(1 for v in recent if v == 0) / len(recent),
           "trend_slope": np.polyfit(range(len(recent)), recent, 1)[0],
           "max_value": max(series),
           "min_value": min(series),
       }
   ```

3. **Generate labeled training data via LLM consensus.** For a representative sample (500-2000 time series windows), prompt multiple LLMs with the domain context and feature summaries. Each LLM independently classifies each sample as anomalous (1) or normal (0) with a written explanation. Require consensus across models and across multiple trials per model. Discard low-confidence samples. Have human experts review a random subset and refine the prompt if systematic errors appear.

4. **Extract qualitative reasoning patterns.** Collect the textual explanations from Step 3 and prompt the LLM to identify recurring detection patterns: "What qualitative indicators were most predictive of anomaly labels?" This yields natural-language rule seeds like "current value is significantly above the historical mean" or "sudden spike after prolonged zero period."

5. **Generate initial rule prototypes as Python functions.** Prompt the LLM with: the qualitative patterns from Step 4, the feature schema from Step 2, 10-20 representative labeled examples, and an instruction to produce an executable Python function `detect_anomaly(features: dict) -> int` using IF-THEN logic with numeric thresholds.

6. **Iteratively optimize rules against labeled data.** Evaluate each rule's precision, recall, and F1 on the labeled set. Compute the confusion matrix. Classify the rule's behavior: over-conservative (low recall, missing true anomalies) or over-aggressive (low precision, too many false positives). Feed this analysis back to the LLM with the current rule code and ask for targeted modifications. Maintain a trajectory log of past rule versions and their scores to prevent cyclic edits. Repeat for 5-15 iterations or until F1 plateaus.

   ```
   Iteration 1: F1=0.72 (over-aggressive, precision=0.61)
   Iteration 2: F1=0.79 (tightened threshold, precision=0.78)
   Iteration 3: F1=0.85 (added z_score condition, recall=0.88)
   ...
   Iteration 8: F1=0.91 (converged)
   ```

7. **Run multi-start optimization.** Repeat Steps 5-6 three to five times with different random example subsets or different initial prompts. Select the best-performing rule set on a held-out test split (not the training data used for optimization).

8. **Augment rules with anomaly categories.** Present the final rule set to the LLM and ask it to analyze each rule's detection principle. Map rules to business-relevant anomaly categories with descriptive names. Extend the detection function to return both a binary label and a category string.

9. **Package rules as a production module.** Export the final rules as a standalone Python module with no LLM dependency. Include: the feature computation function, the rule functions, the category mapping, and a simple `detect(series) -> {"is_anomaly": bool, "category": str, "triggered_rule": str}` entry point.

10. **Validate and deploy.** Run the rule module against a fresh validation set. Confirm F1, precision, and recall meet thresholds. Deploy as a lightweight service or batch job. Log triggered rules and categories for ongoing monitoring. Schedule periodic re-evaluation against new labeled samples.

## Concrete Examples

**Example 1: Supply Chain Inventory Anomaly Detection**

User: "I have weekly out-of-stock rate data for 50,000 products. I need to detect anomalous spikes automatically. Our team currently reviews dashboards manually but can't scale."

Approach:
1. Define domain context: "An anomaly is a sudden, unexplained increase in out-of-stock rate that deviates from the product's historical pattern. Gradual seasonal increases are normal. A decrease in OOS rate is always good, never anomalous."
2. Compute features for each product-week: z-score, rolling mean, ratio to previous week, trend slope.
3. Sample 1,000 product-week windows, label via LLM consensus with the domain context.
4. Generate rules iteratively, optimizing F1.
5. Augment with categories: "Critical Stock-Out Crisis," "Moderate Stock Pressure," "Seasonal Adjustment."

Output rule:
```python
def detect_anomaly(features: dict) -> dict:
    # Rule 1: Extreme spike
    if features["current_value"] >= 80.0:
        return {"is_anomaly": True, "category": "Critical Stock-Out Crisis", "rule": "R1"}
    # Rule 2: Sharp relative increase with elevated level
    if features["ratio_to_prev"] > 10 and features["current_value"] >= 50:
        return {"is_anomaly": True, "category": "Critical Stock-Out Crisis", "rule": "R2"}
    # Rule 3: Statistical outlier with moderate level
    if features["current_value"] >= 35 and features["z_score"] >= 5.0:
        return {"is_anomaly": True, "category": "Moderate Stock Pressure", "rule": "R3"}
    # Rule 4: Sudden emergence from low baseline
    if features["recent_mean"] < 3.0 and features["current_value"] >= 12:
        return {"is_anomaly": True, "category": "Escalating Stock-Out Emergency", "rule": "R4"}
    return {"is_anomaly": False, "category": None, "rule": None}
```

**Example 2: Server Latency Monitoring**

User: "Build anomaly detection for our API latency metrics. We have p99 latency sampled every minute. Alert fatigue is a big problem -- our current z-score threshold fires too often."

Approach:
1. Domain context: "Anomalous latency is a sustained elevation or spike that impacts user experience, not brief jitter. Latency naturally increases during deployments (ignore 10-min windows around deploy timestamps). Weekend traffic patterns differ from weekday."
2. Features: current p99, 15-min rolling mean, 1-hour rolling mean, z-score vs. 24h history, rate of change, time-of-day indicator, is_deploy_window flag.
3. Label 800 samples via LLM with deploy schedules and traffic context included.
4. Optimize rules. The LLM learns to exclude deploy windows and weight sustained elevation over momentary spikes.

Output rule:
```python
def detect_latency_anomaly(features: dict) -> dict:
    if features["is_deploy_window"]:
        return {"is_anomaly": False, "category": None, "rule": "deploy_exclusion"}
    if features["rolling_15m_mean"] > 2.0 * features["rolling_1h_mean"] and features["z_score_24h"] > 3.5:
        return {"is_anomaly": True, "category": "Sustained Latency Degradation", "rule": "L1"}
    if features["current_p99"] > 5000 and features["rate_of_change"] > 200:
        return {"is_anomaly": True, "category": "Acute Latency Spike", "rule": "L2"}
    return {"is_anomaly": False, "category": None, "rule": None}
```

**Example 3: Security Log Anomaly Detection**

User: "Detect anomalous patterns in failed login attempt counts per user per hour. We want to catch credential stuffing without flagging users who just forgot their password."

Approach:
1. Domain context: "Normal: 1-3 failed attempts followed by success. Anomalous: sustained high failure rate from distributed IPs, or sudden spike in a previously inactive account."
2. Features: failed_count, success_after_failure (bool), unique_ip_count, hour_of_day, z_score vs. user's 30-day baseline, account_age_days.
3. Label, optimize, categorize into "Credential Stuffing Suspect," "Brute Force Attempt," "Account Takeover Risk."

Output:
```python
def detect_auth_anomaly(features: dict) -> dict:
    if features["failed_count"] >= 20 and features["unique_ip_count"] >= 5:
        return {"is_anomaly": True, "category": "Credential Stuffing Suspect", "rule": "A1"}
    if features["failed_count"] >= 10 and not features["success_after_failure"] and features["z_score_30d"] > 4.0:
        return {"is_anomaly": True, "category": "Brute Force Attempt", "rule": "A2"}
    if features["account_age_days"] > 365 and features["failed_count"] >= 5 and features["hour_of_day"] in range(1, 6):
        return {"is_anomaly": True, "category": "Account Takeover Risk", "rule": "A3"}
    return {"is_anomaly": False, "category": None, "rule": None}
```

## Best Practices

- **Do:** Write the domain context document first. Rule quality depends entirely on how precisely anomalies are defined in business terms. Spend time getting this right before touching any code.
- **Do:** Use multi-model consensus for labeling (at least 2 different LLM providers). Single-model labeling inherits that model's blind spots.
- **Do:** Track the full optimization trajectory (rule version, F1, precision, recall per iteration). This prevents cyclic edits and provides an audit trail.
- **Do:** Always evaluate on a held-out test set separate from the optimization set. Rules can overfit to the training distribution.
- **Do:** Keep rules as simple Python functions with no external dependencies. The entire point is deterministic, low-latency execution without LLM calls at inference time.
- **Avoid:** Using the LLM for feature computation. Statistical features (z-scores, rolling means) must be computed in deterministic code -- LLMs introduce variance.
- **Avoid:** Optimizing for more than 15 iterations. If F1 hasn't converged by then, the feature set is likely insufficient -- add new features rather than refining thresholds.
- **Avoid:** Deploying rules without category augmentation. Binary anomaly labels alone force operators to re-diagnose every alert. Categories enable automated routing and prioritized response.

## Error Handling

- **Low consensus during labeling:** If LLMs frequently disagree on labels, the domain context is ambiguous. Tighten the definition, add concrete boundary examples, and re-run. Do not force consensus by majority vote alone -- disagreement signals a specification problem.
- **Rules converge to trivial solutions:** If the optimizer produces `return 1` (flag everything) or `return 0` (flag nothing), the class imbalance is too extreme. Stratify the training sample to include at least 15-20% anomalies. If real anomaly rate is lower, oversample anomalies in the training set and validate on the natural distribution.
- **Feature leakage across time:** Ensure rolling statistics are computed causally (only using past data). Future-looking features produce rules that fail in production.
- **Rules don't transfer across subpopulations:** If products/servers/users have fundamentally different baselines, segment them and learn separate rule sets per segment rather than forcing one universal rule.
- **Optimization oscillates without improving:** The trajectory log reveals repeated threshold swings. Instruct the LLM to try structural changes (add/remove conditions) rather than threshold adjustments.

## Limitations

- Rules are symbolic and threshold-based -- they cannot capture complex nonlinear temporal dependencies that deep learning models handle (e.g., subtle shape-based anomalies in multivariate time series).
- The labeling stage still requires human oversight for prompt refinement. Fully automated labeling without expert review will propagate systematic biases.
- Rule sets grow harder to maintain beyond 15-20 rules. For domains requiring hundreds of rules, consider a rule management framework with version control and automated regression testing.
- This approach assumes stationarity within segments. For rapidly evolving data distributions, rules need periodic re-optimization (monthly or quarterly re-runs of the pipeline).
- The technique is best suited for univariate or low-dimensional multivariate time series. High-dimensional sensor arrays may need dimensionality reduction before rule learning.

## Reference

Zhang, H. & Jain, S. (2026). *LLM-Assisted Logic Rule Learning: Scaling Human Expertise for Time Series Anomaly Detection.* arXiv:2601.19255. Key sections: Algorithm 1 (iterative rule optimization loop), Table II (learned rule examples), and the two-tier consensus labeling protocol.