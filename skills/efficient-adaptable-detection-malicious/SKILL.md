---
name: "efficient-adaptable-detection-malicious"
description: >
  Build lightweight, incrementally updatable malicious prompt detection systems using BAGEL's
  bootstrap-aggregated ensemble of small specialized classifiers with random forest routing.
  Use when: "build a prompt safety classifier", "detect jailbreak attempts", "create a
  moderation pipeline for LLM inputs", "implement guardrails for prompt injection",
  "design an ensemble for malicious prompt detection", "add new attack detection without retraining".
---

# BAGEL: Bootstrap-Aggregated Ensemble for Malicious Prompt Detection

This skill enables Claude to design and implement production-grade malicious prompt detection
systems following the BAGEL (Bootstrap AGgregated Ensemble Layer) architecture. Instead of
relying on a single large guardrail model, BAGEL trains multiple small classifiers (86M
parameters each, e.g., Prompt Guard 2), each specialized on a different attack dataset, then
routes incoming prompts through a random forest router and aggregates predictions from a
stochastically selected subset. The result matches or exceeds billion-parameter guardrails
(F1 0.92 vs OpenAI Moderation API's 0.208) at a fraction of the cost, and new attack types
can be added incrementally without retraining the full ensemble.

## When to Use

- When a user needs to build a prompt safety / moderation layer for an LLM-powered application
- When designing guardrails that must detect jailbreaks, prompt injections, and harmful requests
- When the user wants a modular detection system where new attack types can be added without retraining everything
- When replacing or supplementing black-box moderation APIs (OpenAI Moderation, Perspective) with a self-hosted solution
- When the user requires interpretable routing decisions (which attack category triggered, why)
- When computational budget is limited and billion-parameter guardrail models are too expensive
- When building a CI/CD pipeline for prompt safety that must evolve as new attacks emerge

## Key Technique

**Bootstrap-aggregated specialist ensemble.** BAGEL decomposes the broad problem of malicious
prompt detection into narrow sub-problems. Each "promptcop" is a small transformer classifier
(Prompt Guard 2, 86M params) fine-tuned on a single attack dataset — one specialist for
synthetic prompt injections, another for in-the-wild jailbreaks, another for mixed injection
techniques, and so on. This specialization means each model becomes highly accurate on its
attack category without suffering from distribution conflicts across heterogeneous datasets.

**Random forest routing + stochastic selection.** At inference time, nine lightweight structural
features are extracted from the input prompt: `prompt_length`, `whitespace_proportion`,
`special_char_proportion`, `avg_word_length`, `digit_proportion`, `uppercase_proportion`,
`code_keyword_count`, `nl_word_count`, and `shannon_entropy`. A random forest classifier
trained on calibration data predicts which promptcop is the best match. BAGEL then selects
that predicted-best model plus (n-1) additional models chosen uniformly at random, runs all n
models, and averages their maliciousness probabilities. This combines targeted expertise with
ensemble diversity — selecting just 5 of 9 members (430M total params) achieves peak F1.

**Incremental updates without full retraining.** When a new attack type appears, you fine-tune
one new promptcop on the new dataset, append its calibration samples to the global calibration
set, retrain only the lightweight random forest router, and recalibrate the decision threshold.
The existing promptcops remain untouched. After nine incremental updates, performance remains
robust with no degradation.

## Step-by-Step Workflow

1. **Curate specialized attack datasets.** Collect or identify distinct datasets covering
   different attack categories: synthetic prompt injections, jailbreak prompts, mixed-technique
   injections, benign/malicious classification, toxic content. Split each 70% train / 10%
   calibration / 20% test.

2. **Fine-tune one promptcop per dataset.** Start from a small pretrained prompt-safety
   classifier (Prompt Guard 2 or DeBERTa-v3-base, ~86M params). Fine-tune on each dataset's
   training split for 3 epochs using binary cross-entropy or energy-based loss. Validate on the
   held-out 20% of training data. Save each specialist model separately.

3. **Build the calibration set.** Combine the 10% calibration splits from all datasets into a
   single global calibration set. Each sample should have its ground-truth label and be
   associated with the dataset (promptcop index) it came from.

4. **Extract structural features for routing.** For every sample in the calibration set, compute
   the nine features: `prompt_length`, `whitespace_proportion`, `special_char_proportion`,
   `avg_word_length`, `digit_proportion`, `uppercase_proportion`, `code_keyword_count`,
   `nl_word_count`, and `shannon_entropy`. Label each sample with the index of the promptcop
   that achieves the highest confidence on it.

5. **Train the random forest router.** Fit a `RandomForestClassifier` on the structural
   features (X) with promptcop indices as targets (y). This router predicts which specialist
   is most relevant for a given prompt based purely on surface-level text statistics — no
   tokenization or embedding required, so routing is sub-millisecond.

6. **Calibrate the decision threshold.** Run BAGEL inference (router + stochastic ensemble of
   n=5) over the entire calibration set. Perform a coarse-to-fine grid search (20 evaluations)
   over threshold values to find `tau*` that maximizes F1 score.

7. **Implement the inference pipeline.** For each incoming prompt: (a) extract 9 structural
   features, (b) query the random forest to get predicted best promptcop index `i*`, (c) select
   `{M_i*} ∪ {4 randomly sampled others}`, (d) run all 5 models, (e) compute
   `y_hat = (1/5) * sum(p_j(x))`, (f) classify as malicious if `y_hat > tau*`.

8. **Add interpretability output.** Log the router's predicted category, the feature importances
   from the random forest, and each promptcop's individual score. This reveals *why* a prompt
   was flagged — e.g., "routed to jailbreak specialist due to high shannon_entropy and
   code_keyword_count."

9. **Implement incremental update for new attacks.** When a new attack dataset arrives:
   fine-tune a new promptcop, append calibration samples, retrain the random forest, recalibrate
   the threshold. No existing models are modified. Wrap this in a script or CI job.

10. **Deploy with fallback and monitoring.** Serve the ensemble behind an API. Monitor per-promptcop
    trigger rates, router confidence, and threshold margin distribution. Set alerts for
    distribution drift that might signal a novel attack category not covered by existing specialists.

## Concrete Examples

**Example 1: Building a BAGEL ensemble from scratch**

User: "I need to build a prompt safety classifier for my chatbot. I have datasets of jailbreak
prompts, prompt injections, and normal user queries. How should I architect this?"

Approach:
1. Structure datasets into three specialist training sets, each with 70/10/20 splits.
2. Fine-tune Prompt Guard 2 three times (once per dataset), producing three promptcops.
3. Compute structural features for calibration samples:

```python
import math
import re
from collections import Counter

KEYWORDS = {"import", "exec", "eval", "system", "os", "subprocess", "sudo", "rm", "drop", "select"}

def extract_features(prompt: str) -> dict:
    tokens = prompt.split()
    chars = list(prompt)
    char_counts = Counter(chars)
    total_chars = len(prompt) or 1
    total_words = len(tokens) or 1

    # Shannon entropy over characters
    entropy = -sum((c / total_chars) * math.log2(c / total_chars)
                    for c in char_counts.values() if c > 0)

    return {
        "prompt_length": len(prompt),
        "whitespace_proportion": sum(1 for c in chars if c.isspace()) / total_chars,
        "special_char_proportion": sum(1 for c in chars if not c.isalnum() and not c.isspace()) / total_chars,
        "avg_word_length": sum(len(w) for w in tokens) / total_words,
        "digit_proportion": sum(1 for c in chars if c.isdigit()) / total_chars,
        "uppercase_proportion": sum(1 for c in chars if c.isupper()) / total_chars,
        "code_keyword_count": sum(1 for w in tokens if w.lower() in KEYWORDS),
        "nl_word_count": total_words,
        "shannon_entropy": entropy,
    }
```

4. Train the random forest router and calibrate threshold:

```python
from sklearn.ensemble import RandomForestClassifier
import numpy as np

# X_cal: (N, 9) feature matrix from calibration set
# y_best_cop: (N,) index of promptcop with highest confidence per sample
router = RandomForestClassifier(n_estimators=100, random_state=42)
router.fit(X_cal, y_best_cop)

# Threshold calibration via coarse-to-fine search
best_tau, best_f1 = 0.5, 0.0
for tau in np.linspace(0.1, 0.9, 20):
    preds = (ensemble_scores_cal > tau).astype(int)
    f1 = f1_score(y_true_cal, preds)
    if f1 > best_f1:
        best_tau, best_f1 = tau, f1
```

5. Inference returns a label plus interpretability metadata.

**Example 2: Incrementally adding a new attack detector**

User: "A new prompt injection technique just appeared. I have 2,000 labeled examples. How do I
update my BAGEL system without retraining everything?"

Approach:
1. Split the 2,000 samples: 1,400 train / 200 calibration / 400 test.
2. Fine-tune a new promptcop from the Prompt Guard 2 base on the 1,400 training samples (3 epochs).
3. Append the 200 calibration samples (with new promptcop index) to the global calibration set.
4. Retrain the random forest on the expanded calibration features.
5. Re-run coarse-to-fine threshold search on the full updated calibration set.
6. Deploy — existing promptcops are completely unchanged.

```python
# Pseudocode for incremental update
new_cop = fine_tune_prompt_guard2(new_train_data, epochs=3)
ensemble.append(new_cop)

# Expand calibration
new_cal_features = [extract_features(s) for s in new_cal_data]
global_cal_X = np.vstack([global_cal_X, new_cal_features])
global_cal_y = np.concatenate([global_cal_y, [len(ensemble) - 1] * len(new_cal_data)])

# Retrain router (lightweight, seconds)
router.fit(global_cal_X, global_cal_y)

# Recalibrate threshold
tau_star = calibrate_threshold(ensemble, global_cal_set)
```

**Example 3: Inference pipeline with interpretability**

User: "Show me how the runtime detection works for a single prompt."

```python
def detect(prompt: str, ensemble, router, tau: float, n: int = 5) -> dict:
    features = extract_features(prompt)
    feat_vec = np.array(list(features.values())).reshape(1, -1)

    # Route to best specialist
    best_idx = router.predict(feat_vec)[0]

    # Stochastic selection: best + (n-1) random others
    others = [i for i in range(len(ensemble)) if i != best_idx]
    selected = [best_idx] + random.sample(others, min(n - 1, len(others)))

    # Run selected promptcops
    scores = {i: ensemble[i].predict_proba(prompt) for i in selected}
    avg_score = np.mean(list(scores.values()))

    return {
        "malicious": avg_score > tau,
        "score": round(avg_score, 4),
        "threshold": tau,
        "routed_to": best_idx,
        "individual_scores": scores,
        "routing_features": features,
        "feature_importances": dict(zip(features.keys(),
                                        router.feature_importances_)),
    }
```

Output:
```json
{
  "malicious": true,
  "score": 0.8734,
  "threshold": 0.52,
  "routed_to": 2,
  "individual_scores": {2: 0.95, 0: 0.81, 5: 0.88, 7: 0.79, 3: 0.94},
  "routing_features": {"prompt_length": 487, "shannon_entropy": 4.82, "code_keyword_count": 6, "...": "..."},
  "feature_importances": {"shannon_entropy": 0.23, "prompt_length": 0.18, "...": "..."}
}
```

## Best Practices

- **Do:** Keep each promptcop specialized on exactly one attack dataset. Cross-dataset training
  defeats the purpose of the mixture-of-experts decomposition.
- **Do:** Use all nine structural features for routing. The feature set was designed to capture
  distinct attack surface signals (entropy for obfuscation, code keywords for injection,
  uppercase proportion for social engineering emphasis).
- **Do:** Select n=5 ensemble members at inference. The paper shows this is the sweet spot —
  fewer loses coverage, more adds latency without meaningful F1 gain.
- **Do:** Re-run threshold calibration after every incremental update. The optimal threshold
  shifts as the ensemble composition changes.
- **Avoid:** Using the router alone without stochastic selection. The random additional members
  provide critical ensemble diversity that catches edge cases the router misclassifies.
- **Avoid:** Fine-tuning promptcops on combined/merged datasets. Each specialist must see only
  its attack category to maintain the bootstrap aggregation benefit.

## Error Handling

- **Router misclassification:** The stochastic selection of additional members mitigates this.
  If monitoring shows the router consistently misrouting a new prompt pattern, it signals that
  a new promptcop may be needed for that pattern.
- **Threshold drift:** After incremental updates, always recalibrate. If F1 degrades on the
  global calibration set, inspect per-promptcop accuracy to identify a degraded specialist.
- **Imbalanced datasets:** Some attack datasets are tiny (1,042 samples) while others are huge
  (464,000). Use stratified sampling and consider oversampling the calibration set for small
  datasets so the router gets adequate signal.
- **Feature extraction failures:** Guard against empty prompts (division by zero in proportions)
  and extremely long prompts (truncate to a max length before feature extraction).
- **Cold start with few attack types:** BAGEL needs at least 3 promptcops for stochastic
  selection to be meaningful. With fewer, fall back to running all available specialists.

## Limitations

- BAGEL's structural features are surface-level text statistics. Semantically sophisticated
  attacks that look statistically identical to benign prompts may evade the router (though
  individual promptcops may still catch them via transformer understanding).
- Each promptcop requires a separate fine-tuning run and model checkpoint. At 86M parameters
  each, 9 models total ~770M params on disk. This is lightweight compared to a single 2B+
  guardrail but not negligible for edge deployment.
- The random forest router is trained on calibration data distribution. Adversarial feature
  manipulation (deliberately matching benign feature profiles) could fool routing, though the
  stochastic selection provides partial defense.
- The approach requires labeled attack datasets. For truly novel zero-day prompt attacks with
  no examples, BAGEL cannot train a specialist until samples are collected.
- Inference latency scales linearly with n (number of selected members). At n=5 with 86M
  models, this is fast on GPU but may be a consideration for CPU-only deployments.

## Reference

**Paper:** [Efficient and Adaptable Detection of Malicious LLM Prompts via Bootstrap Aggregation](https://arxiv.org/abs/2602.08062v1) (Hassan et al., 2026)
**Code:** [github.com/sands-lab/bagel](https://github.com/sands-lab/bagel)
**Key insight:** An ensemble of 5 small specialized classifiers (430M params total) with
random forest routing outperforms billion-parameter guardrails by decomposing prompt safety
into per-attack-type sub-problems and aggregating via bootstrap selection.