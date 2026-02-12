---
name: "linglanmidian-systematic-evaluation-tcm"
description: "Build rigorous, multi-task evaluation benchmarks for domain-specific LLMs using the LingLanMiDian methodology: synonym-tolerant matching, difficulty-ranked hard subsets, character-level F1, and decision recognition reframing. Trigger phrases: 'evaluate LLM on domain knowledge', 'build a medical benchmark', 'TCM evaluation pipeline', 'synonym-tolerant scoring', 'create hard subset for benchmark', 'domain-specific LLM evaluation'"
---

# LingLanMiDian: Systematic Domain-Specific LLM Evaluation

This skill enables Claude to design and implement rigorous, multi-task evaluation benchmarks for domain-specific LLMs, applying the LingLanMiDian methodology from TCM (Traditional Chinese Medicine) evaluation. The core techniques—synonym-tolerant bipartite matching, composite-difficulty hard subset selection, character-level F1 scoring, embedding-based distractor generation for decision recognition, and unified multi-format metric design—generalize to any specialized domain where terminology is rich, answers have multiple valid surface forms, and you need fair, reproducible model comparison.

## When to Use

- When the user needs to evaluate LLMs on a specialized domain (medicine, law, finance, engineering) and existing benchmarks are fragmented or generation-heavy
- When building a scoring pipeline that must tolerate synonyms, abbreviations, or near-equivalent labels (e.g., "MI" vs "myocardial infarction")
- When the user wants to construct a "hard subset" from an existing benchmark to stress-test model robustness
- When converting open-ended clinical/domain tasks (diagnosis, recommendation) into single-choice format for standardized scoring
- When implementing character-level or token-level F1 for partial-credit scoring on cloze/fill-in-the-blank tasks
- When designing extraction metrics that handle multiset entity matching with multiplicity
- When the user asks to compare multiple LLMs fairly across heterogeneous task formats (MCQ, cloze, extraction, open-ended)

## Key Technique

**Unified multi-format evaluation with synonym tolerance.** LingLanMiDian's central insight is that domain-specific evaluation fails when you apply a single metric to heterogeneous task types, or when exact-match scoring penalizes semantically correct answers that differ in surface form. The benchmark defines five metric families—accuracy for classification, instance-level multiset P/R/F1 for extraction, character-level F1 for cloze, synonym-tolerant bipartite F1 (DTR-F1) for open-ended clinical labels, and cosine similarity + MAE for quantitative predictions—then applies each consistently within its task category across all datasets.

**Synonym-tolerant bipartite matching (DTR-F1).** For open-ended tasks where multiple valid labels exist (e.g., syndrome names), the algorithm constructs a bipartite graph between predicted labels and gold labels. An edge exists between prediction y_hat and gold y if their character-level F1 exceeds threshold tau (default 0.7). Maximum-cardinality matching on this graph yields true positives, with unmatched predictions as false positives and unmatched golds as false negatives. This enforces one-to-one alignment while tolerating near-synonymous forms—critical in any domain with rich terminological variation.

**Composite-difficulty hard subset construction.** Rather than hand-picking hard examples, LingLan scores each item by running all N models and computing D = (1 - mu) + lambda * sigma, where mu is mean accuracy across models and sigma is variance. High D means items that are both hard on average and discriminative (high variance implies some models get it right, others don't). The top-400 items per task form the Hard subset. This is a reusable technique for any benchmark: it surfaces items that differentiate models rather than items that are uniformly impossible.

## Step-by-Step Workflow

1. **Define the task taxonomy.** Enumerate the domain's evaluation dimensions: knowledge recall (MCQ), reasoning (multi-hop MCQ), information extraction (NER/span), cloze completion, open-ended generation, and quantitative prediction. Map each to a specific metric family before writing any code.

2. **Standardize data format.** Represent every item as a JSON object with fields: `id`, `task_type` (enum), `question`, `options` (nullable), `gold_answer` (string or list), `metadata` (source, difficulty tags). For extraction tasks, `gold_answer` is a list of `{entity, type, count}` objects. For dosage tasks, include a `gold_vector` of `{herb, dose_grams}` pairs.

3. **Implement the metric registry.** Build a dispatcher that selects the scoring function based on `task_type`:
   - `single_choice` → Accuracy (exact match on option letter)
   - `multi_choice` → Instance-level accuracy + option-level Precision/Recall/F1
   - `cloze` → Character-level F1 (treat strings as character multisets)
   - `extraction` → Multiset P/R/F1 with type+normalized-form equality
   - `open_label` → DTR-F1 with synonym-tolerant bipartite matching (tau=0.7)
   - `quantitative` → Cosine similarity on aligned dose vectors + MAE

4. **Implement synonym-tolerant matching.** For each instance: (a) compute pairwise character-F1 between every predicted label and every gold label, (b) build a bipartite graph keeping edges where F1 >= tau, (c) run the Hopcroft-Karp or Hungarian algorithm for maximum-cardinality matching, (d) derive TP/FP/FN from match counts.

5. **Implement character-level F1.** For two strings s and s_hat: count character occurrences as multisets. TP = sum of min(count_s(c), count_s_hat(c)) for each character c. FP = sum of max(0, count_s_hat(c) - count_s(c)). FN = sum of max(0, count_s(c) - count_s_hat(c)). Compute P = TP/(TP+FP), R = TP/(TP+FN), F1 = 2PR/(P+R).

6. **Convert open-ended tasks to decision recognition.** For open-ended diagnosis/treatment tasks: (a) embed all candidate labels with a sentence encoder, (b) for each test case, retrieve the top-K nearest neighbors to the gold label, (c) construct a single-choice item with the gold as correct and K-1 semantically close distractors. This transforms generation evaluation into classification, enabling accuracy-based comparison.

7. **Run zero-shot evaluation.** Query each model with standardized prompts, temperature=0.6, max_tokens=8192. Parse structured responses. Do not fine-tune or few-shot—the goal is measuring intrinsic domain capability.

8. **Construct the Hard subset.** After collecting all model predictions: for each item, compute mu (mean accuracy across models) and sigma (variance). Score D = (1 - mu) + 0.5 * sigma. Rank items descending by D within each task. Select top-N (e.g., 400) per task.

9. **Report dual performance.** For every model and every task, report both Full-set and Hard-subset scores side by side. The gap (Full minus Hard) reveals robustness. Macro-average across tasks for the headline score.

10. **Validate with human baselines.** Have domain experts complete a sample of the Hard subset to establish a human ceiling. Report the model-to-human gap as the primary indicator of remaining research distance.

## Concrete Examples

**Example 1: Building a synonym-tolerant scorer for medical NER**

User: "I have a TCM NER dataset where models predict syndrome names, but they use different surface forms—e.g., '气滞血瘀' vs '气滞血瘀证'. How do I score this fairly?"

Approach:
1. Normalize both gold and predicted labels (strip trailing classifiers like '证'/'型' if desired, but the algorithm handles this automatically).
2. For each test instance, compute pairwise char-F1 between all predicted and gold labels.
3. Build bipartite graph with edges where char-F1 >= 0.7.
4. Run maximum-cardinality matching.
5. Compute instance-level P/R/F1 from match counts, then macro-average.

```python
from collections import Counter
from scipy.optimize import linear_sum_assignment
import numpy as np

def char_f1(s1: str, s2: str) -> float:
    """Character-level F1 between two strings treated as char multisets."""
    c1, c2 = Counter(s1), Counter(s2)
    tp = sum((c1 & c2).values())
    fp = sum((c1 - c2).values())  # in predicted but not gold
    fn = sum((c2 - c1).values())  # in gold but not predicted
    if tp == 0:
        return 0.0
    p = tp / (tp + fp)
    r = tp / (tp + fn)
    return 2 * p * r / (p + r)

def synonym_tolerant_f1(predicted: list[str], gold: list[str], tau: float = 0.7):
    """DTR-F1: bipartite matching with char-F1 threshold."""
    n, m = len(predicted), len(gold)
    if n == 0 and m == 0:
        return 1.0, 1.0, 1.0
    if n == 0 or m == 0:
        return 0.0, 0.0, 0.0

    # Cost matrix (negative F1 for minimization)
    cost = np.full((n, m), 1e9)
    for i, p in enumerate(predicted):
        for j, g in enumerate(gold):
            f = char_f1(p, g)
            if f >= tau:
                cost[i, j] = -f

    row_ind, col_ind = linear_sum_assignment(cost)
    tp = sum(1 for r, c in zip(row_ind, col_ind) if cost[r, c] < 1e9)
    fp = n - tp
    fn = m - tp
    precision = tp / (tp + fp) if (tp + fp) > 0 else 0.0
    recall = tp / (tp + fn) if (tp + fn) > 0 else 0.0
    f1 = 2 * precision * recall / (precision + recall) if (precision + recall) > 0 else 0.0
    return precision, recall, f1
```

Output: A scorer that gives credit for "气滞血瘀" matching "气滞血瘀证" (char-F1 ≈ 0.89 > 0.7) while rejecting "肝气郁结" as a match (char-F1 ≈ 0.25 < 0.7).

---

**Example 2: Constructing a difficulty-ranked Hard subset**

User: "I have a legal domain benchmark with 5,000 items and results from 10 models. I want to find the 200 hardest items that also discriminate between models."

Approach:
1. Load per-item binary correctness vectors across all 10 models.
2. Compute per-item mean accuracy (mu) and variance (sigma).
3. Score each item: D = (1 - mu) + 0.5 * sigma.
4. Sort descending by D, take top 200.

```python
import numpy as np

def build_hard_subset(
    item_ids: list[str],
    correctness: np.ndarray,  # shape (n_items, n_models), binary
    top_k: int = 200,
    lambda_weight: float = 0.5
) -> list[str]:
    """Select hard, discriminative items using LingLan difficulty scoring."""
    mu = correctness.mean(axis=1)       # mean accuracy per item
    sigma = correctness.var(axis=1)     # variance per item
    difficulty = (1 - mu) + lambda_weight * sigma
    ranked_indices = np.argsort(-difficulty)[:top_k]
    return [item_ids[i] for i in ranked_indices]

# Example: item with mu=0.1, sigma=0.09 → D = 0.9 + 0.045 = 0.945 (hard + discriminative)
# Item with mu=0.0, sigma=0.0 → D = 1.0 + 0.0 = 1.0 (hard but not discriminative—all fail)
# Item with mu=0.5, sigma=0.25 → D = 0.5 + 0.125 = 0.625 (moderate but very discriminative)
```

Output: A ranked list of 200 item IDs forming the Hard subset, biased toward items that are both difficult and informative for distinguishing model capabilities.

---

**Example 3: Reframing open-ended diagnosis as decision recognition**

User: "Our evaluation has open-ended syndrome diagnosis questions. Models generate free text, making comparison unreliable. How do I convert this to single-choice?"

Approach:
1. Build an embedding index of all candidate syndrome labels in the domain ontology.
2. For each test case, embed the gold syndrome label.
3. Retrieve the top-K (e.g., 1000) nearest neighbors by cosine similarity.
4. Sample 3 distractors from the neighbors, preferring labels that are semantically close but incorrect.
5. Construct a 4-option MCQ (1 correct + 3 distractors). Shuffle option order.

```python
from sentence_transformers import SentenceTransformer
import numpy as np

def build_decision_recognition_items(
    cases: list[dict],           # each has 'question', 'gold_label'
    all_labels: list[str],       # full label ontology
    model_name: str = "BAAI/bge-base-zh-v1.5",
    n_distractors: int = 3,
    top_k: int = 1000
) -> list[dict]:
    """Convert open-ended diagnosis to single-choice decision recognition."""
    encoder = SentenceTransformer(model_name)
    label_embeddings = encoder.encode(all_labels, normalize_embeddings=True)

    items = []
    for case in cases:
        gold_emb = encoder.encode([case["gold_label"]], normalize_embeddings=True)
        sims = (gold_emb @ label_embeddings.T).flatten()
        top_indices = np.argsort(-sims)[:top_k]

        # Filter: exclude exact gold match, pick closest distractors
        distractors = []
        for idx in top_indices:
            if all_labels[idx] != case["gold_label"]:
                distractors.append(all_labels[idx])
            if len(distractors) == n_distractors:
                break

        options = [case["gold_label"]] + distractors
        np.random.shuffle(options)
        correct_idx = options.index(case["gold_label"])

        items.append({
            "question": case["question"],
            "options": {chr(65 + i): opt for i, opt in enumerate(options)},
            "gold_answer": chr(65 + correct_idx)
        })
    return items
```

Output: Each open-ended case becomes a 4-option MCQ where distractors are semantically close (e.g., for gold "肝郁脾虚", distractors might be "肝郁气滞", "脾虚湿盛", "肝脾不调"), enabling accuracy-based scoring.

## Best Practices

- **Do** set the synonym-tolerance threshold tau based on your domain. In TCM, tau=0.7 works because Chinese medical terms share characters across syndromes. In English medical text, you may need tau=0.6 to tolerate abbreviation differences. Calibrate by checking a sample of true synonyms and near-misses.

- **Do** report both Full and Hard subset scores for every model. The gap between them is often more informative than either score alone—it reveals brittleness on edge cases.

- **Do** use character-level F1 (not token-level) for CJK languages where tokenization is inconsistent across models. For English domains, token-level (word-level) F1 is the natural analog.

- **Avoid** using generation-heavy metrics (BLEU, ROUGE) for clinical label evaluation. They conflate fluency with correctness. The bipartite matching approach evaluates semantic correctness directly.

- **Avoid** hand-picking hard items based on intuition. The composite-difficulty formula D = (1 - mu) + lambda * sigma is data-driven and reproducible. Items with high variance are especially valuable because they reveal what distinguishes stronger models.

- **Do** enforce one-to-one matching in the synonym-tolerant protocol. Without it, a single predicted label could match multiple gold labels (or vice versa), inflating scores. The bipartite matching constraint prevents this.

## Error Handling

- **No predictions parsed:** If a model returns unparseable output for a task, score it as TP=0, FP=0, FN=|gold| (full miss). Log the parse failure rate per model—high rates indicate prompt format issues, not domain weakness.

- **Empty gold label set:** Skip items with empty gold annotations in metric computation. Flag them for data quality review.

- **Threshold sensitivity:** If small changes to tau (e.g., 0.65 vs 0.75) cause large score swings, your label set has many borderline synonyms. Address this upstream with label normalization, or report results at multiple tau values.

- **Degenerate distractors in decision recognition:** If embedding retrieval yields distractors that are too easy (low similarity) or identical to the gold (data duplication), add a similarity band filter: keep distractors with cosine similarity in [0.5, 0.95] relative to the gold.

- **Imbalanced task sizes:** When macro-averaging across tasks of different sizes, ensure each task contributes equally to the headline score regardless of item count. Per-task scores should be computed first, then averaged.

## Limitations

- **Synonym tolerance is not semantic equivalence.** Character-level F1 at tau=0.7 catches surface-form variations but misses true synonyms with completely different characters (e.g., "心悸" vs "心慌"). For such cases, integrate an external synonym dictionary or use embedding-based similarity as the matching criterion instead of char-F1.

- **Hard subset stability.** The difficulty ranking depends on which models are evaluated. Adding or removing a model changes mu and sigma, potentially reshuffling the Hard subset. Pin the model set when constructing the subset for reproducible comparisons.

- **Decision recognition simplifies the task.** Converting open-ended generation to single-choice makes evaluation cleaner but easier—a model might select the correct label from options but fail to generate it unprompted. Report both DR accuracy and open-ended DTR-F1 for completeness.

- **Zero-shot only.** The methodology evaluates intrinsic domain knowledge without few-shot or RAG augmentation. If your use case involves retrieval, the benchmark results may not predict deployed performance.

- **CJK-specific character F1.** The character-level multiset approach works naturally for Chinese (each character carries meaning). For morphologically rich languages (German, Finnish), subword-level F1 may be more appropriate than raw character F1.

## Reference

**Paper:** [LingLanMiDian: Systematic Evaluation of LLMs on TCM Knowledge and Clinical Reasoning](https://arxiv.org/abs/2602.01779v1) (Hua et al., 2026). Focus on Section 3 (Metric Design) for the synonym-tolerant protocol and character-level F1 formulas, Section 2.3 for Hard subset construction, and Section 2.2 for decision recognition conversion. Code and data at [github.com/TCMAI-BJTU/LingLan](https://github.com/TCMAI-BJTU/LingLan).