---
name: "borp-bootstrapped-regression-probing"
description: >
  Build scalable LLM evaluation pipelines that measure user satisfaction from hidden states
  rather than generative scoring. Implements the BoRP framework: polarization-index bootstrapping
  for automatic rubric generation, PLS regression over latent representations, and CUPED-based
  A/B testing with variance reduction. Trigger phrases: "evaluate LLM satisfaction",
  "probe hidden states for quality", "build a BoRP pipeline", "score conversations without
  LLM-as-judge", "set up satisfaction A/B testing", "automated rubric generation".
---

# BoRP: Bootstrapped Regression Probing for Scalable LLM Evaluation

This skill enables Claude to build production evaluation pipelines that score conversational AI satisfaction by probing LLM hidden states with regression, rather than using expensive generative LLM-as-judge approaches. The core insight from the BoRP paper (Sun, Zhang & Wu, 2026) is that latent space geometry already encodes quality signals -- you extract hidden-state vectors, reduce them with Partial Least Squares (PLS) regression, and get continuous satisfaction scores that correlate with human judgments better than models 10x larger used as generative judges, at orders-of-magnitude lower cost.

## When to Use

- When the user wants to evaluate conversational AI quality at scale without paying for generative judge calls on every conversation
- When the user needs to run highly sensitive A/B tests on chatbot changes and traditional metrics (thumbs up/down, implicit signals) are too sparse or noisy
- When the user asks to build an automated rubric system that discovers what quality dimensions matter for their domain
- When the user wants to monitor LLM output quality in production with continuous scoring rather than binary pass/fail
- When the user needs to replace or supplement LLM-as-judge pipelines with a faster, cheaper, more human-aligned alternative
- When the user asks about CUPED variance reduction for online experiments on conversational products

## Key Technique

**Why hidden-state probing beats generative judging:** Generative evaluation (prompting an LLM to rate a conversation) is expensive and often poorly calibrated -- a small model asked to "rate this from 1-10" produces noisy ordinal outputs. BoRP instead treats the LLM as a feature extractor. By reading the hidden-state activations from intermediate transformer layers while the model processes a conversation, you get a dense vector that geometrically encodes quality properties the model has learned. A lightweight PLS regression head maps these vectors to continuous satisfaction scores. Because PLS explicitly finds latent components that maximize covariance between hidden states and human labels, it cuts through the curse of dimensionality (hidden states are 4096+ dimensions) while preserving the signal that matters.

**Polarization-index bootstrapping for rubric generation:** Instead of hand-crafting evaluation rubrics, BoRP generates them automatically. The polarization index measures how much a candidate rubric dimension separates "good" from "bad" conversations in latent space. The bootstrapping loop works like this: (1) sample a subset of labeled conversations, (2) generate candidate rubric dimensions via prompting, (3) compute the polarization index for each dimension by measuring how well it splits the hidden-state clusters, (4) retain high-polarization dimensions and discard noisy ones. After several bootstrap iterations, you have a rubric that is empirically grounded in what the model's internal representations actually distinguish.

**CUPED for A/B testing:** Once you have continuous per-conversation satisfaction scores, you can run A/B tests with dramatically higher sensitivity. CUPED (Controlled-experiment Using Pre-Experiment Data) uses each user's historical satisfaction scores as a control variate, subtracting out baseline variance. Because BoRP scores are continuous (not binary thumbs-up/down), CUPED reduces variance far more effectively, letting you detect smaller treatment effects with the same sample size.

## Step-by-Step Workflow

1. **Prepare labeled conversation data.** Collect a dataset of at least 500-2000 conversations with human satisfaction labels (Likert scale, pairwise preference, or continuous scores). Split into train (70%), validation (15%), and test (15%). Store as JSONL with fields: `conversation_id`, `turns` (list of role/content dicts), `satisfaction_score` (float 0-1).

2. **Select the probe model and extraction layers.** Choose an open-weight LLM (Qwen3-8B or 14B are validated; Llama-3 and Mistral work too). Identify extraction layers -- start with the last quarter of layers (e.g., layers 24-32 for a 32-layer model). Use SGLang or vLLM for efficient batched inference with hidden-state output enabled.

3. **Extract hidden-state vectors.** For each conversation, concatenate all turns into a single prompt (or use the chat template), run a forward pass, and extract the hidden-state vector at the last token position from each selected layer. Average across selected layers to produce one vector per conversation. Save as a NumPy matrix of shape `(N, hidden_dim)`.

4. **Run polarization-index bootstrapping to generate rubrics.** For each bootstrap iteration: (a) sample 60% of labeled conversations, (b) prompt the LLM to propose 10-20 candidate quality dimensions (e.g., "factual accuracy", "tone appropriateness", "completeness"), (c) for each candidate dimension, compute a polarization index -- the ratio of between-cluster variance to total variance when conversations are split by that dimension in hidden-state space, (d) retain dimensions with polarization index above the 75th percentile. After 10-20 bootstrap rounds, take dimensions that survived in >50% of rounds as the final rubric.

5. **Score conversations on each rubric dimension.** Use the probe LLM to assign per-dimension scores. Construct prompts that present the conversation and ask for a 1-5 score on each rubric dimension. Collect these as a feature matrix alongside the hidden states.

6. **Train PLS regression from hidden states to satisfaction.** Fit a PLS model (scikit-learn's `PLSRegression`) with 5-20 components mapping the hidden-state matrix to the satisfaction scores. Use cross-validation on the training set to select the component count that minimizes MAE on validation data. This is a fast operation -- PLS training on 2000 samples with 4096 features takes seconds.

7. **Evaluate alignment with human judgments.** On the held-out test set, compute Spearman rank correlation, Pearson correlation, and MAE between predicted and actual satisfaction scores. A well-tuned BoRP pipeline should achieve Spearman > 0.65 on typical conversational data. Compare against a generative baseline (prompting a large model to directly score satisfaction) to quantify the improvement.

8. **Deploy for production scoring.** Wrap the pipeline into a service: (a) receive conversation JSON, (b) run forward pass through the probe LLM to extract hidden states, (c) apply the trained PLS model to produce a satisfaction score, (d) log the score. Because there is no generation step, throughput is 10-100x higher than generative judging.

9. **Integrate CUPED for A/B testing.** For each user in an experiment, compute their pre-experiment average satisfaction score (from production logs). During the experiment, compute the CUPED-adjusted metric: `Y_cuped = Y - theta * X_pre`, where `theta = Cov(Y, X_pre) / Var(X_pre)`. This removes baseline user-level variance, increasing the t-statistic for the same sample size.

10. **Monitor and recalibrate.** Set up weekly checks: re-extract hidden states on a fresh sample, compare PLS predictions against a small batch of new human labels. If Spearman drops below 0.55, retrain the PLS model. If the rubric polarization indices shift, re-run bootstrapping.

## Concrete Examples

**Example 1: Building a satisfaction scorer for a customer support chatbot**

User: "We have 5000 labeled support conversations with CSAT scores. I want to score all future conversations automatically without using GPT-4 as a judge."

Approach:
1. Load the 5000 conversations, normalize CSAT to 0-1 range, split 70/15/15
2. Deploy Qwen3-8B via SGLang with `--return-hidden-states` flag
3. Extract hidden states from layers 24-32, average to get 4096-dim vectors
4. Run 15 bootstrap rounds to discover rubric (typical output: "resolution completeness", "empathy", "response accuracy", "follow-up appropriateness")
5. Train PLS with 10 components, validated MAE ~0.08 on held-out set
6. Deploy as a FastAPI endpoint that accepts conversation JSON and returns a score

Output:
```python
# Core scoring function after training
import numpy as np
from sklearn.cross_decomposition import PLSRegression
import joblib

# Load trained model
pls_model = joblib.load("borp_pls_model.pkl")

def score_conversation(hidden_states: np.ndarray) -> float:
    """Map extracted hidden states to satisfaction score."""
    # hidden_states shape: (1, 4096) -- averaged across layers
    score = pls_model.predict(hidden_states.reshape(1, -1))[0, 0]
    return float(np.clip(score, 0.0, 1.0))

# Example result
# Input: support conversation about billing dispute
# Predicted satisfaction: 0.73 (actual human CSAT: 0.75)
# Spearman correlation on test set: 0.71
```

**Example 2: Running a CUPED-enhanced A/B test on a new system prompt**

User: "We changed the system prompt for our assistant and want to measure if satisfaction improved. We have 2 weeks of pre-experiment BoRP scores per user."

Approach:
1. Collect pre-experiment BoRP scores per user (X_pre = mean of last 14 days)
2. Run experiment for 7 days, computing BoRP scores for control and treatment
3. Apply CUPED adjustment to reduce noise from user-level baseline differences
4. Run two-sample t-test on adjusted scores

Output:
```python
import numpy as np
from scipy import stats

def cuped_analysis(y_control, y_treat, x_pre_control, x_pre_treat):
    """CUPED-adjusted A/B test using BoRP satisfaction scores."""
    # Compute theta from pooled data
    y_all = np.concatenate([y_control, y_treat])
    x_all = np.concatenate([x_pre_control, x_pre_treat])
    theta = np.cov(y_all, x_all)[0, 1] / np.var(x_all)

    # Adjust scores
    y_ctrl_adj = y_control - theta * x_pre_control
    y_treat_adj = y_treat - theta * x_pre_treat

    # Run t-test
    t_stat, p_value = stats.ttest_ind(y_treat_adj, y_ctrl_adj)
    effect_size = y_treat_adj.mean() - y_ctrl_adj.mean()

    variance_reduction = 1 - np.var(y_ctrl_adj) / np.var(y_control)
    return {
        "effect_size": effect_size,
        "p_value": p_value,
        "variance_reduction_pct": variance_reduction * 100,
    }

# Example result:
# Raw t-test p-value: 0.12 (not significant)
# CUPED-adjusted p-value: 0.003 (significant!)
# Variance reduction: 42%
# Effect size: +0.04 satisfaction points
```

**Example 3: Automated rubric discovery via polarization-index bootstrapping**

User: "I don't know what quality dimensions matter for our coding assistant. Can you discover the rubric automatically?"

Approach:
1. Sample 60% of labeled conversations, extract hidden states
2. Prompt the LLM to propose candidate dimensions for coding assistant quality
3. For each dimension, split conversations into high/low groups and compute between-group variance in hidden-state space
4. Repeat 15 times with different samples, retain dimensions surviving >50% of rounds

Output:
```
Discovered rubric after 15 bootstrap rounds:
  Dimension                  | Polarization Index | Survival Rate
  ---------------------------|--------------------|---------------
  Code correctness           | 0.82               | 100%
  Explanation clarity        | 0.71               | 93%
  Edge case handling         | 0.64               | 80%
  Conciseness                | 0.58               | 73%
  API/library accuracy       | 0.55               | 67%
  ---------------------------|--------------------|---------------
  Formatting style (dropped) | 0.23               | 20%
  Greeting tone (dropped)    | 0.15               | 7%

Final rubric: 5 dimensions retained (polarization > 0.50, survival > 50%)
```

## Best Practices

- **Do:** Extract hidden states from multiple layers in the last quarter of the model and average them. Single-layer extraction is brittle; averaging provides robustness to layer-specific noise.
- **Do:** Normalize satisfaction labels to a continuous 0-1 range before PLS training. PLS is sensitive to scale, and bounded targets improve generalization.
- **Do:** Use at least 500 human-labeled conversations for training. Below this threshold, PLS overfits and Spearman correlation degrades sharply.
- **Do:** Re-run polarization bootstrapping when you change the probe model or shift to a new domain. Rubric dimensions that matter for customer support differ from those for coding assistants.
- **Avoid:** Using generative scoring (prompting the probe model to output a number) alongside BoRP. The two signals are redundant and the generative pass eliminates BoRP's cost advantage.
- **Avoid:** Extracting hidden states from early layers (first 25% of the model). These layers encode syntactic/positional features, not quality-relevant semantics.
- **Avoid:** Setting PLS components above 20 without cross-validation evidence. Over-parameterized PLS memorizes training artifacts and the satisfaction scores lose calibration on new data.

## Error Handling

- **Hidden-state extraction returns NaN or zeros:** Check that the inference framework is configured to return hidden states (SGLang: `--return-logprob` alone is insufficient; you need `--return-hidden-states`). Verify the layer indices are within the model's range.
- **PLS model produces scores outside 0-1:** Apply `np.clip(score, 0, 1)` at prediction time. If most predictions cluster near 0 or 1, the training labels may not be properly normalized -- check for outliers in the label distribution.
- **Spearman correlation below 0.50 on validation:** First verify label quality (inter-annotator agreement should be >0.60). Then try increasing bootstrap rounds to 20+ to improve rubric quality. If still poor, try a larger probe model (14B instead of 8B).
- **CUPED adjustment increases variance instead of reducing it:** This happens when pre-experiment scores have low correlation with experiment-period scores (e.g., user behavior shifted). Fall back to unadjusted analysis or use a shorter pre-experiment window.
- **Polarization index is uniformly low across all candidate dimensions:** The labeled dataset may lack sufficient diversity. Ensure the training data includes a balanced spread across satisfaction levels, not just mostly-good or mostly-bad conversations.

## Limitations

- Requires access to model hidden states, so it only works with open-weight models where you control inference. API-only models (GPT-4, Claude) cannot be probed this way.
- The PLS regression head is domain-specific. A model trained on customer support conversations will not transfer well to coding assistant evaluation without retraining.
- BoRP scores are relative to the training distribution. If conversation quality shifts substantially (new product launch, model upgrade), the PLS model needs recalibration against fresh human labels.
- The polarization-index bootstrapping assumes that human satisfaction is at least partially encoded in the probe model's latent space. For tasks far outside the probe model's pretraining distribution, this assumption may not hold.
- This is a pre-print (January 2026). The technique has been validated on industrial datasets at the authors' organization but independent replication is still pending.

## Reference

[BoRP: Bootstrapped Regression Probing for Scalable and Human-Aligned LLM Evaluation](https://arxiv.org/abs/2601.18253v1) -- Sun, Zhang & Wu, 2026. Focus on Section 3 (polarization-index bootstrapping algorithm), Section 4 (PLS regression training procedure), and Section 5.3 (CUPED integration for A/B testing sensitivity gains).