---
name: "linglanmidian-systematic-evaluation-tcm"
description: "Build rigorous, multi-task LLM evaluation benchmarks for specialized domains using LingLanMiDian's methodology: synonym-tolerant scoring, difficulty-stratified hard subsets, distractor generation via embedding retrieval, and decision-recognition reframing. Triggers: 'evaluate LLM on domain knowledge', 'build a benchmark for specialized reasoning', 'create synonym-tolerant scoring', 'TCM evaluation benchmark', 'domain-specific LLM evaluation suite', 'hard subset difficulty scoring'"
---

This skill teaches Claude to design and implement systematic, multi-task evaluation benchmarks for specialized domains (medical, legal, cultural) using the LingLanMiDian methodology. The core techniques include: constructing synonym-tolerant matching protocols that handle terminological variation, generating difficulty-scored Hard subsets from model disagreement signals, reframing open-ended clinical/expert tasks as discriminative single-choice problems with embedding-retrieved distractors, and applying task-appropriate metric hierarchies (character-level F1, list-level precision/recall, cosine similarity for continuous outputs). These techniques generalize well beyond TCM to any domain where terminology varies, expert reasoning is multi-step, and fair cross-model comparison matters.

## When to Use

- When the user asks to build an evaluation benchmark for a specialized domain (medicine, law, finance, classical literature) where terminology is non-standardized
- When the user needs to compare multiple LLMs on domain-specific knowledge and reasoning tasks with fair, consistent metrics
- When the user wants to create difficulty-stratified test sets that expose gaps between models and human experts
- When the user needs to convert open-ended generation tasks (diagnosis, recommendation) into reproducible single-choice evaluations
- When the user asks to implement synonym-tolerant or fuzzy matching for evaluating LLM outputs against gold labels
- When the user needs to generate high-quality distractors for multiple-choice evaluation items using embedding similarity
- When the user is building a TCM (Traditional Chinese Medicine) knowledge evaluation pipeline

## Key Technique

**Unified multi-task evaluation with synonym tolerance.** LingLanMiDian's central insight is that domain-specific LLM evaluation fails when tasks use inconsistent scoring or when surface-form variation in expert terminology causes false negatives. The benchmark solves this with a bipartite matching protocol: predicted and gold label sets are treated as nodes in a bipartite graph, edges connect pairs whose character-level F1 exceeds a threshold (tau=0.7), and maximum-cardinality matching determines true positives. This allows "blood stasis" to match "blood stagnation" without a manually curated synonym dictionary, while still rejecting genuinely wrong answers.

**Difficulty scoring via model disagreement.** Rather than relying on human difficulty ratings (expensive and subjective), LingLan computes item difficulty as `D = (1 - mu) + lambda * sigma`, where `mu` is the mean accuracy across all evaluated models and `sigma` is the variance. High difficulty means most models fail (low mu) and they disagree about it (high sigma). The top 400 items per task form the Hard subset, which reliably exposes the gap between frontier models and domain experts.

**Decision recognition via embedding-retrieved distractors.** Open-ended clinical tasks (syndrome differentiation, treatment recommendation) are notoriously hard to score fairly. LingLan reframes them as single-choice questions: the correct answer is the gold-standard clinical label, and distractors are sourced by encoding the case with a lightweight embedding model, retrieving the top-K nearest neighbors, and selecting semantically close but incorrect options. This yields discriminative items that test genuine clinical reasoning rather than surface-level pattern matching.

## Step-by-Step Workflow

1. **Define the domain taxonomy and task types.** Enumerate the evaluation dimensions for your domain (e.g., knowledge recall, multi-step reasoning, information extraction, decision-making). Map each dimension to concrete task formats: single-choice, multi-choice, cloze/fill-in-blank, structured extraction, or decision recognition.

2. **Curate or source gold-standard items with expert review.** Collect items from licensing exams, textbooks, clinical records, or domain corpora. Ensure each item has: a question/prompt, a gold answer (possibly a set of labels), the task type, and a domain category. Have domain experts validate at least a sample.

3. **Assign task-appropriate metrics to each subtask.**
   - Single-choice accuracy for factual recall
   - Instance-level and option-level precision/recall/F1 for multi-label tasks
   - Character-level F1 (multiset intersection over character multiplicities) for cloze/short-answer
   - List-level precision/recall/F1 with bipartite matching for extraction tasks
   - MAE and cosine similarity for continuous-valued predictions (e.g., dosage, quantities)

4. **Implement the synonym-tolerant matching protocol.** For each (predicted_label, gold_label) pair, compute character-level F1. Build a bipartite graph with edges where F1 >= tau (default 0.7). Run maximum-cardinality matching (e.g., Hopcroft-Karp) to determine TP. Compute precision = TP / |predicted|, recall = TP / |gold|, F1 from these.

   ```python
   from collections import Counter
   import networkx as nx

   def char_f1(pred: str, gold: str) -> float:
       p_chars, g_chars = Counter(pred), Counter(gold)
       overlap = sum((p_chars & g_chars).values())
       if overlap == 0:
           return 0.0
       precision = overlap / sum(p_chars.values())
       recall = overlap / sum(g_chars.values())
       return 2 * precision * recall / (precision + recall)

   def synonym_tolerant_match(preds: list[str], golds: list[str], tau: float = 0.7) -> tuple[float, float, float]:
       G = nx.Graph()
       for i, p in enumerate(preds):
           for j, g in enumerate(golds):
               if char_f1(p, g) >= tau:
                   G.add_edge(f"p_{i}", f"g_{j}")
       matching = nx.max_weight_matching(G, maxcardinality=True)
       tp = len(matching)
       precision = tp / len(preds) if preds else 0.0
       recall = tp / len(golds) if golds else 0.0
       f1 = 2 * precision * recall / (precision + recall) if (precision + recall) > 0 else 0.0
       return precision, recall, f1
   ```

5. **Generate difficulty scores and select the Hard subset.** Run all candidate models on the full benchmark. For each item, compute `mu` (mean correctness across models) and `sigma` (variance). Score difficulty as `D = (1 - mu) + 0.5 * sigma`. Rank items within each task by D descending. Select the top N (e.g., 400) per task.

   ```python
   import numpy as np

   def compute_difficulty(item_results: list[list[bool]], top_n: int = 400) -> list[int]:
       """item_results[i][j] = True if model j got item i correct."""
       scores = []
       for results in item_results:
           mu = np.mean(results)
           sigma = np.var(results)
           d = (1 - mu) + 0.5 * sigma
           scores.append(d)
       ranked = np.argsort(scores)[::-1]
       return ranked[:top_n].tolist()
   ```

6. **Reframe open-ended tasks as decision recognition with embedding-retrieved distractors.** For each item requiring a clinical/expert judgment (e.g., "What syndrome does this case present?"), encode the case and all candidate labels using a lightweight embedding model. Retrieve the top-K nearest labels by cosine similarity. Use the gold label as the correct answer and select 3-4 semantically close but incorrect labels as distractors.

   ```python
   from sentence_transformers import SentenceTransformer
   import numpy as np

   def generate_distractors(case_text: str, gold_label: str, all_labels: list[str],
                            model_name: str = "Qwen/Qwen3-0.6B", top_k: int = 1000,
                            n_distractors: int = 3) -> list[str]:
       model = SentenceTransformer(model_name)
       case_emb = model.encode([case_text])
       label_embs = model.encode(all_labels)
       sims = np.dot(label_embs, case_emb.T).flatten()
       ranked_idx = np.argsort(sims)[::-1][:top_k]
       distractors = []
       for idx in ranked_idx:
           if all_labels[idx] != gold_label and len(distractors) < n_distractors:
               distractors.append(all_labels[idx])
       return distractors
   ```

7. **Configure zero-shot evaluation parameters.** Set consistent decoding hyperparameters across all models: temperature T=0.6, max generation length 8192 tokens, enable reasoning/thinking mode where supported. Do not apply task-specific prompt engineering to ensure fair comparison.

8. **Run evaluation and compute per-task and aggregate scores.** Execute each model against all items. Parse outputs into structured predictions (choice letter, label set, numeric value). Apply the correct metric per task type. Report instance-level macro-averaged scores to ensure equal task weighting.

9. **Analyze Full vs. Hard subset performance.** Compare model rankings on the full set and the Hard subset. A large drop on the Hard subset signals brittleness in domain reasoning. Use this gap as the primary signal for identifying where models need domain-specific improvement.

10. **Generate the evaluation report with leaderboard.** Produce a table with per-task scores and overall averages for both Full and Hard subsets. Highlight tasks where the best model still falls substantially below expert baselines. Export raw per-item results for downstream error analysis.

## Concrete Examples

**Example 1: Building a synonym-tolerant NER evaluator for medical records**

User: "I have an LLM extracting symptom entities from clinical notes. The model outputs 'chest tightness' but the gold label says 'chest oppression'. How do I score this fairly?"

Approach:
1. Implement character-level F1 between predicted and gold entity strings
2. Set threshold tau=0.7 for matching tolerance
3. Build bipartite graph across all predicted and gold entity sets per instance
4. Run maximum-cardinality matching to count true positives
5. Compute precision, recall, and F1 from the matching

Output:
```
Predicted: ["chest tightness", "headache", "fatigue"]
Gold:      ["chest oppression", "headache", "lassitude"]

char_f1("chest tightness", "chest oppression") = 0.72  >= 0.7 -> MATCH
char_f1("headache", "headache") = 1.0              >= 0.7 -> MATCH
char_f1("fatigue", "lassitude") = 0.25              < 0.7 -> NO MATCH

TP=2, Precision=2/3=0.67, Recall=2/3=0.67, F1=0.67
```

**Example 2: Creating a Hard subset from multi-model evaluation results**

User: "I evaluated 8 LLMs on 2000 legal reasoning questions. How do I identify the hardest questions to make a challenging test set?"

Approach:
1. Collect binary correctness matrix (2000 items x 8 models)
2. For each item compute mu (mean accuracy) and sigma (variance)
3. Score difficulty D = (1 - mu) + 0.5 * sigma
4. Rank by D descending and take top 400

Output:
```python
item_results = [
    [True, False, False, False, True, False, False, False],  # Item 0: mu=0.25, sigma=0.1875, D=0.844
    [True, True, True, True, True, True, True, False],       # Item 1: mu=0.875, sigma=0.109, D=0.180
    [False, False, False, True, False, False, False, False],  # Item 2: mu=0.125, sigma=0.109, D=0.930
]
# Item 2 ranks hardest (most models fail, some disagreement)
# Item 0 ranks next (low accuracy, high variance)
# Item 1 ranks easiest (most models succeed)
hard_subset_indices = compute_difficulty(item_results, top_n=400)
```

**Example 3: Converting open-ended diagnosis to single-choice with embedding distractors**

User: "Our TCM evaluation has a case study asking 'What is the syndrome differentiation?' Currently it's scored with exact match and models score near 0%. How do I make this evaluable?"

Approach:
1. Compile a label inventory of all valid syndrome names from the training corpus
2. Encode the case description with a sentence embedding model
3. Retrieve top-1000 similar syndrome labels by cosine similarity
4. Use the gold syndrome as answer A; pick 3 high-similarity but incorrect syndromes as B, C, D
5. Present as single-choice; score with accuracy

Output:
```
Case: "Male, 45, presents with distending pain in the hypochondrium,
       bitter taste, dry throat, string-like rapid pulse..."

Gold syndrome: "Liver-Gallbladder Damp-Heat"

Retrieved distractors (by embedding similarity to case):
  B. Liver Qi Stagnation          (sim=0.82)
  C. Spleen-Stomach Damp-Heat     (sim=0.79)
  D. Liver Fire Flaming Upward    (sim=0.76)

Question: What is the primary syndrome differentiation?
A. Liver-Gallbladder Damp-Heat  B. Liver Qi Stagnation
C. Spleen-Stomach Damp-Heat     D. Liver Fire Flaming Upward

This converts a 0% exact-match task into a discriminative 40-60% accuracy task
that meaningfully differentiates model clinical reasoning ability.
```

## Best Practices

- **Do:** Use character-level F1 (not token-level or exact match) for synonym tolerance -- it naturally handles partial overlap in domain terminology across languages, especially Chinese where character-level decomposition is semantically meaningful.
- **Do:** Compute difficulty from multi-model disagreement rather than human judgments -- it is cheaper, more reproducible, and directly measures what separates models.
- **Do:** Apply instance-level macro-averaging across tasks so that each evaluation dimension contributes equally regardless of dataset size differences.
- **Do:** Keep decoding parameters (temperature, max tokens) consistent across all models to ensure fair comparison; avoid task-specific prompt tuning in benchmarking contexts.
- **Avoid:** Using generation-heavy scoring (BLEU, ROUGE) for clinical evaluation -- these correlate poorly with domain correctness and penalize valid reformulations.
- **Avoid:** Setting the synonym-tolerance threshold tau too low (< 0.5) -- this risks matching genuinely different clinical concepts. The paper's tau=0.7 balances tolerance and precision.
- **Avoid:** Selecting distractors randomly from the label space -- semantically distant distractors make items trivially easy. Always use embedding-based retrieval to ensure distractors are plausible and discriminative.

## Error Handling

- **Parsing failures:** When LLM output does not conform to expected format (e.g., no clear choice letter), implement a fallback regex cascade: first match `[A-D]`, then search for the full option text, then mark as unanswered. Log unparseable responses for manual review.
- **Empty predictions in extraction tasks:** Treat as zero TP, full FN. Do not skip these items -- they represent meaningful model failures.
- **Embedding model unavailability:** If the specified embedding model is not accessible, fall back to TF-IDF cosine similarity for distractor retrieval. Results will be noisier but still produce semantically relevant distractors.
- **Tied difficulty scores:** When multiple items share the same D score at the Hard subset boundary, break ties by lower mu (harder items first), then by higher sigma (more discriminative items).
- **Threshold sensitivity:** If synonym-tolerant matching produces unexpected results, visualize the character-F1 distribution between true matches and false matches. Adjust tau to the valley between the two distributions.

## Limitations

- **Character-level F1 is language-dependent.** The synonym-tolerant protocol works well for Chinese (where characters carry semantic weight) and reasonably for English, but may need adaptation for agglutinative languages (Turkish, Finnish) or languages with complex morphology.
- **Difficulty scoring requires multiple models.** The Hard subset method needs evaluation results from at least 5-8 diverse models to produce stable difficulty estimates. With fewer models, sigma becomes noisy.
- **Decision recognition reframing trades recall for precision.** Converting open-ended tasks to single-choice eliminates the ability to detect novel or creative correct answers that were not in the distractor pool. It measures recognition, not generation.
- **Zero-shot evaluation underestimates fine-tuned models.** The benchmark methodology is designed for comparing general-purpose models. Domain-fine-tuned models may need few-shot or instruction-tuned evaluation formats to show their true capability.
- **Distractor quality depends on label inventory size.** If the domain has fewer than ~100 candidate labels, embedding-based retrieval may not find sufficiently close distractors, reducing item discriminability.

## Reference

**Paper:** Hua, R., Wei, Y., Shu, Z., Chang, K., & Yan, D. (2026). *LingLanMiDian: Systematic Evaluation of LLMs on TCM Knowledge and Clinical Reasoning.* arXiv:2602.01779v1. [https://arxiv.org/abs/2602.01779v1](https://arxiv.org/abs/2602.01779v1)

Look for: Section 3 (benchmark construction methodology), Section 4 (synonym-tolerant protocol and metric design), and the appendix for prompt templates and full task specifications. Code and data at [https://github.com/TCMAI-BJTU/LingLan](https://github.com/TCMAI-BJTU/LingLan).