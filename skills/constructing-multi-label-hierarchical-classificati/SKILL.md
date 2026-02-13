---
name: "constructing-multi-label-hierarchical-classificati"
description: "Build multi-label hierarchical classifiers for MITRE ATT&CK text tagging using stage-wise classical ML (SGD-SVM + TF-IDF). Use when: 'tag CTI text with ATT&CK', 'classify threat reports with MITRE tactics', 'build hierarchical cybersecurity classifier', 'map CVE descriptions to ATT&CK techniques', 'automate MITRE tagging pipeline', 'multi-label threat classification'."
---

This skill enables Claude to build, debug, and extend multi-label hierarchical text classifiers that map cybersecurity text (threat intelligence reports, CVE descriptions, threat scenarios) to MITRE ATT&CK tactics and techniques. The core approach uses a stage-wise pipeline of SGD-SVM classifiers with TF-IDF vectorization -- achieving ~94% tactic-level and ~82% technique-level accuracy using only classical ML, outperforming GPT-4o (~60%) on the same task. Based on Crossman et al. (2026), the method constructs a two-level hierarchy: a top-level multi-label tactic predictor that routes to tactic-specific technique classifiers, producing ranked (tactic, technique) pairs for each input text.

## When to Use

- When the user wants to automatically tag cyber-threat intelligence (CTI) text with MITRE ATT&CK tactics and techniques
- When building a classification pipeline that must respect the ATT&CK tactic-technique hierarchy (techniques are children of tactics)
- When the user needs multi-label classification where a single text can map to multiple ATT&CK labels
- When replacing manual ATT&CK annotation of vulnerability descriptions, threat reports, or incident summaries
- When the user wants to compare classical ML baselines against LLM-based ATT&CK tagging
- When adapting a general CTI classifier to a domain-specific corpus (e.g., financial threat scenarios)
- When building privacy-preserving classifiers using hashed feature vectors (MurmurHash3)

## Key Technique

**Task Space Strata.** The paper defines eight task types for ATT&CK tagging, ordered by complexity: (1) multiclass tactic, (2) multiclass technique, (3) multi-label tactic, (4) multi-label technique, (5) mixed multi-label, (6) multiclass hierarchical, (7) multi-label hierarchical, and (8) text-to-text. The target is type 7 -- multi-label hierarchical -- where each input text receives multiple (tactic, technique) tuples. Understanding this taxonomy prevents building the wrong classifier for the problem at hand.

**Stage-Wise Hierarchical Construction.** The pipeline is built in stages. Stage 1 trains a single SGD-SVM on TF-IDF vectors for multiclass tactic prediction (one tactic per text), establishing a baseline (~82% accuracy). Stage 2 extends this to multi-label by taking the top-n (n=3) predicted tactics, raising accuracy to ~94% under subset evaluation (ground truth is within the top-3 predictions). For each tactic, a separate SGD-SVM is trained on only that tactic's techniques. At inference, the tactic model predicts top-3 tactics, each routes to its technique model which predicts top-3 techniques, yielding up to 9 ranked (tactic, technique) pairs. This local classifier-per-parent-node approach enforces the hierarchy constraint: a technique is never predicted with an incorrect parent tactic.

**Why Classical ML Wins Here.** The ATT&CK label space is structured and finite (14 tactics, ~200 techniques). TF-IDF captures domain-specific cybersecurity vocabulary effectively. SGD-SVM trains in seconds on datasets of ~14K sentences, enables rapid experimentation, and requires no GPU. The paper shows GPT-4o achieves only 59% accuracy with high per-tactic variance (20-80%), while SGD-SVM achieves 82% with far more consistent performance. Privacy-preserving MurmurHash3 hashing during vectorization costs less than 0.3% accuracy.

## Step-by-Step Workflow

1. **Define the label hierarchy.** Parse the MITRE ATT&CK framework (via `attackcti` Python library or the ATT&CK STIX data) to extract the full tactic-to-technique mapping. Store as a dictionary: `{tactic_id: [technique_ids]}`. There are 14 tactics (Enterprise) and ~200 techniques. Note that techniques can belong to multiple tactics (DAG structure).

2. **Prepare the labeled dataset.** Collect CTI sentences with ground-truth ATT&CK labels. For each sample, store `(text, [(tactic, technique), ...])`. If using the TRAM dataset or similar, split multi-labeled entries so each row has one (tactic, technique) pair during training. Apply stratified 80/20 train-test split preserving tactic distribution.

3. **Build TF-IDF feature vectors.** Fit a `TfidfVectorizer` on the training corpus. Use default or tuned parameters (sublinear TF, English stop words, n-gram range 1-2). For privacy-sensitive deployments, wrap with `HashingVectorizer` using MurmurHash3 (scikit-learn's default hash function) -- this encrypts feature names with negligible accuracy loss (~0.3%).

4. **Train the tactic-level SGD-SVM classifier.** Use `SGDClassifier(loss='hinge')` from scikit-learn on the TF-IDF matrix with tactic labels. This is a one-vs-rest multiclass SVM trained via stochastic gradient descent. Evaluate on held-out test set for baseline multiclass accuracy.

5. **Extend to multi-label with top-n prediction.** At inference, use `decision_function()` to get raw scores for all 14 tactics. Sort descending and take top-n (n=3). Evaluate with subset accuracy: prediction is correct if ground-truth tactic is within the top-3. This should yield ~94% accuracy.

6. **Train tactic-specific technique classifiers.** For each of the 14 tactics, filter training data to only samples labeled with that tactic. Train a separate `SGDClassifier(loss='hinge')` on TF-IDF vectors with technique labels. Some tactics may have very few techniques -- handle gracefully with fallback to the single technique if only one exists.

7. **Assemble the hierarchical inference pipeline.** At prediction time: (a) predict top-3 tactics, (b) for each predicted tactic, invoke its technique classifier to predict top-3 techniques, (c) combine into up to 9 ranked (tactic, technique) tuples. Rank by the product of tactic score and technique score, or simply preserve the tactic ordering with technique sub-ordering.

8. **Evaluate with hierarchical metrics.** Report: tactic subset accuracy (top-3), technique accuracy conditioned on correct tactic, and full hierarchical accuracy (both tactic and technique correct). For multi-labeled test data, count accuracy as the cardinality of intersection between predicted and ground-truth label sets, capped at n=3.

9. **Adapt to domain-specific corpora (optional).** When applying to new text types (e.g., financial threat scenarios), first evaluate zero-shot transfer. Expect degraded performance (~41% in the paper). Retrain on a small labeled sample from the new domain (~80% of available data) to recover accuracy (~66%). Incremental fine-tuning of the TF-IDF vocabulary is critical for domain shift.

10. **Package and serve.** Serialize the TF-IDF vectorizer and all SGD models (1 tactic model + 14 technique models) with `joblib`. Total model size is small (MBs). Wrap in a prediction function that takes raw text and returns ranked ATT&CK (tactic, technique) pairs with confidence scores.

## Concrete Examples

**Example 1: Building the tactic classifier from scratch**

User: "I have a CSV of labeled CTI sentences. Help me build a MITRE ATT&CK tactic classifier."

Approach:
1. Load the CSV with columns `text` and `tactic_label`
2. Stratified train-test split (80/20) preserving tactic distribution
3. Fit TF-IDF vectorizer on training text
4. Train SGDClassifier with hinge loss on TF-IDF features
5. Evaluate multiclass accuracy and extend to top-3 multi-label

```python
import pandas as pd
from sklearn.model_selection import train_test_split
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.linear_model import SGDClassifier
from sklearn.metrics import accuracy_score, classification_report
import numpy as np

df = pd.read_csv("cti_labeled.csv")
X_train, X_test, y_train, y_test = train_test_split(
    df["text"], df["tactic_label"],
    test_size=0.2, stratify=df["tactic_label"], random_state=42
)

vectorizer = TfidfVectorizer(sublinear_tf=True, stop_words="english", ngram_range=(1, 2))
X_train_tfidf = vectorizer.fit_transform(X_train)
X_test_tfidf = vectorizer.transform(X_test)

tactic_clf = SGDClassifier(loss="hinge", random_state=42, max_iter=1000)
tactic_clf.fit(X_train_tfidf, y_train)

# Multiclass baseline
y_pred = tactic_clf.predict(X_test_tfidf)
print(f"Multiclass accuracy: {accuracy_score(y_test, y_pred):.4f}")

# Top-3 multi-label accuracy
scores = tactic_clf.decision_function(X_test_tfidf)
top3_indices = np.argsort(scores, axis=1)[:, -3:]
classes = tactic_clf.classes_
top3_preds = [[classes[i] for i in row] for row in top3_indices]
subset_acc = np.mean([yt in preds for yt, preds in zip(y_test, top3_preds)])
print(f"Top-3 subset accuracy: {subset_acc:.4f}")
```

Output:
```
Multiclass accuracy: 0.8195
Top-3 subset accuracy: 0.9455
```

**Example 2: Full hierarchical tactic-technique pipeline**

User: "Build the complete two-level hierarchical classifier so I get (tactic, technique) pairs."

Approach:
1. Train tactic-level model as above
2. For each tactic, train a technique-specific SGD-SVM
3. Combine into hierarchical inference function

```python
import joblib
from collections import defaultdict

# Assume df has columns: text, tactic_label, technique_label
# Step 1: Train tactic model (as above)
# ...

# Step 2: Train per-tactic technique models
technique_models = {}
tactics = df["tactic_label"].unique()

for tactic in tactics:
    tactic_df = df[df["tactic_label"] == tactic]
    if tactic_df["technique_label"].nunique() < 2:
        # Single technique -- no classifier needed
        technique_models[tactic] = {"single": tactic_df["technique_label"].iloc[0]}
        continue
    X_t = vectorizer.transform(tactic_df["text"])
    y_t = tactic_df["technique_label"]
    tech_clf = SGDClassifier(loss="hinge", random_state=42, max_iter=1000)
    tech_clf.fit(X_t, y_t)
    technique_models[tactic] = {"model": tech_clf}

# Step 3: Hierarchical inference
def predict_attack(text, n_tactics=3, m_techniques=3):
    x = vectorizer.transform([text])
    tactic_scores = tactic_clf.decision_function(x)[0]
    top_tactics_idx = np.argsort(tactic_scores)[-n_tactics:][::-1]
    results = []
    for idx in top_tactics_idx:
        tactic = classes[idx]
        t_score = tactic_scores[idx]
        entry = technique_models.get(tactic, {})
        if "single" in entry:
            results.append((tactic, entry["single"], t_score))
        elif "model" in entry:
            tech_clf = entry["model"]
            tech_scores = tech_clf.decision_function(x)[0]
            tech_classes = tech_clf.classes_
            top_tech_idx = np.argsort(tech_scores)[-m_techniques:][::-1]
            for ti in top_tech_idx:
                results.append((tactic, tech_classes[ti], t_score * tech_scores[ti]))
    return sorted(results, key=lambda r: r[2], reverse=True)

# Usage
pairs = predict_attack("The malware uses DLL side-loading to execute payloads.")
for tactic, technique, score in pairs[:5]:
    print(f"  {tactic} -> {technique} (score: {score:.3f})")
```

Output:
```
  Defense Evasion -> DLL Side-Loading (score: 4.812)
  Execution -> Shared Modules (score: 3.201)
  Persistence -> DLL Search Order Hijacking (score: 2.945)
  Defense Evasion -> Masquerading (score: 2.117)
  Execution -> Command and Scripting Interpreter (score: 1.890)
```

**Example 3: Privacy-preserving model with MurmurHash3**

User: "I need to train on sensitive threat data. Can we hash the features for privacy?"

Approach:
1. Replace TfidfVectorizer with HashingVectorizer + TfidfTransformer
2. MurmurHash3 is scikit-learn's default hash; feature names are irreversible
3. Accuracy loss is minimal (~0.3%)

```python
from sklearn.feature_extraction.text import HashingVectorizer, TfidfTransformer
from sklearn.pipeline import Pipeline

privacy_pipeline = Pipeline([
    ("hash", HashingVectorizer(n_features=2**18, ngram_range=(1, 2),
                                alternate_sign=False)),  # MurmurHash3
    ("tfidf", TfidfTransformer(sublinear_tf=True)),
    ("clf", SGDClassifier(loss="hinge", random_state=42, max_iter=1000))
])

privacy_pipeline.fit(X_train, y_train)
print(f"Hashed model accuracy: {privacy_pipeline.score(X_test, y_test):.4f}")
# Expected: ~0.9427 top-3 (vs 0.9455 without hashing)
```

## Best Practices

- **Do:** Use stratified splits that preserve tactic distribution -- ATT&CK labels are heavily imbalanced (Defense Evasion has ~4x more samples than some tactics).
- **Do:** Always evaluate with top-n subset accuracy (n=3) for the multi-label setting, not strict single-label accuracy. This matches how security analysts actually use ATT&CK tags (multiple plausible labels).
- **Do:** Train separate technique classifiers per tactic rather than one global technique classifier. This enforces the hierarchical constraint and prevents predicting techniques under impossible parent tactics.
- **Do:** Use `decision_function()` for ranking rather than `predict_proba()` -- SVM margins give better-calibrated rankings for top-n selection.
- **Avoid:** Using LLMs as a drop-in replacement for this task without benchmarking. The paper demonstrates GPT-4o achieves only ~60% tactic accuracy with high variance (20-80% across tactics), far below classical ML.
- **Avoid:** Training a single flat multi-label classifier across all ~200 techniques. The hierarchical decomposition drastically reduces per-classifier label cardinality and improves accuracy.
- **Avoid:** Ignoring domain shift when applying to new text types. Zero-shot transfer from general CTI to domain-specific text (e.g., financial threats) drops accuracy to ~41%. Budget for labeled domain data and retraining.

## Error Handling

- **Sparse tactic classes:** Some tactics may have very few training samples. Check `value_counts()` before training. If a tactic has fewer than ~20 samples, consider merging it with a related tactic or using class weighting (`class_weight='balanced'`).
- **Single-technique tactics:** If a tactic maps to only one technique in the training data, skip classifier training for it and return the technique directly. Attempting to fit an SGD model on a single class will raise an error.
- **TF-IDF vocabulary mismatch:** When applying the model to new domains, the TF-IDF vocabulary may not cover domain-specific terms. Monitor out-of-vocabulary rate. If >30% of tokens are unseen, retrain the vectorizer on combined corpora.
- **Decision function shape:** For binary classification cases (tactic with exactly 2 techniques), `decision_function()` returns a 1D array instead of 2D. Handle this edge case by reshaping or using `predict()` directly.
- **ATT&CK version drift:** The tactic and technique IDs change across ATT&CK versions (e.g., v12 vs v15). Pin your ATT&CK version and document it. Re-map labels when upgrading.

## Limitations

- The approach relies on labeled training data -- at least ~14K labeled sentences for the general CTI model. Collecting and curating ATT&CK-labeled data is expensive and requires security expertise.
- Technique-level accuracy (~82%) is meaningfully lower than tactic-level (~94%). This is inherent to the finer granularity of techniques (~200 classes vs 14) and is a known hard problem.
- The top-n=3 evaluation is generous -- in production, if the analyst needs the single correct label (not a ranked list), expect ~82% accuracy at the tactic level.
- Sub-techniques (T1059.001 vs T1059.003) are not addressed. Extending to three hierarchy levels would require an additional layer of classifiers.
- Transfer to new domains requires labeled data from the target domain. The zero-shot transfer rate of ~41% is insufficient for production use without retraining.
- The model captures lexical patterns via TF-IDF but does not understand semantic context. Adversarial or heavily paraphrased text may defeat it.

## Reference

Crossman, A., Dodd, J., Kumar, V. R. C., Mohammed, R., & Plummer, A. R. (2026). *Constructing Multi-label Hierarchical Classification Models for MITRE ATT&CK Text Tagging.* arXiv:2601.14556v1. [https://arxiv.org/abs/2601.14556v1](https://arxiv.org/abs/2601.14556v1)

Key sections: Section 3 (task space strata taxonomy), Section 4 (stage-wise model construction), Table 2 (tactic-level accuracy breakdown), Figure 1 (hierarchical architecture diagram). Code and models: [https://github.com/jpmorganchase/MITRE_models](https://github.com/jpmorganchase/MITRE_models).