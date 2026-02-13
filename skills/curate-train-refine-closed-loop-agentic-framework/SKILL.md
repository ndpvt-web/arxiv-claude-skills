---
name: "curate-train-refine-closed-loop-agentic-framework-"
description: "Build lightweight text classifiers from zero labeled data using an agentic Curate-Train-Refine loop. An LLM generates synthetic training data, trains a small classifier, analyzes its errors, and synthesizes targeted examples to fix weaknesses — iterating until convergence. Use when: 'build a text classifier without labeled data', 'zero-shot classification with a small model', 'generate training data for a classifier', 'distill LLM into lightweight classifier', 'train SetFit model from scratch', 'agentic data curation loop'."
---

# Curate-Train-Refine: Agentic Zero-Shot Classification

This skill enables Claude to implement the Curate-Train-Refine (CTR) closed-loop framework from Maheshwari & El Haddad (2026). Instead of deploying a large LLM for inference-time classification, CTR uses an LLM as a **data curator**: it generates synthetic labeled examples, trains a lightweight classifier (e.g., SetFit at 110M parameters), analyzes that classifier's errors via confusion matrices and error traces, then synthesizes targeted hard examples to patch the weaknesses — repeating until performance plateaus. The result is a small, fast, deployable classifier that rivals or exceeds zero/few-shot LLM baselines, at a fraction of the inference cost.

## When to Use

- When a user needs a text classifier but has **zero or very few labeled examples**
- When the user wants to **distill LLM classification ability** into a small, fast model for production deployment
- When the user asks to **generate synthetic training data** for a text classification task
- When inference latency or cost rules out using an LLM directly for classification at scale
- When the user mentions **SetFit**, contrastive fine-tuning, or wants a sentence-transformer-based classifier
- When the user has class labels and descriptions but no annotated corpus
- When the user wants to iteratively improve a classifier by analyzing its failure modes

## Key Technique

The CTR framework replaces manual data labeling with a three-phase agentic loop. In the **Curate** phase, the LLM generates an initial batch of synthetic training examples for each class, guided by label names, optional descriptions, and constraints on class balance and deduplication. It also creates a held-out synthetic validation set, disjoint from training data. In the **Train** phase, a lightweight classifier (the paper uses SetFit with `all-mpnet-base-v2`, 110M parameters) is trained on the synthetic data and evaluated against the validation set, producing detailed diagnostics: accuracy, macro-F1, confusion matrix, per-class error rates, and example-level error traces.

In the **Refine** phase, the LLM ingests these diagnostics and performs structured error analysis — identifying systematic confusions between class pairs, sensitivity to lexical cues, negation failures, and domain gaps. It then synthesizes **targeted** new examples: hard negatives for confused pairs, boundary cases, paraphrases, and edge cases that address the specific weaknesses found. These examples are added to the training set and the loop repeats. The loop terminates when either a maximum iteration count is reached or validation performance stops improving (no gain exceeding threshold epsilon for p consecutive rounds). A per-class budget caps total generated examples to prevent runaway growth.

The key insight is that **generic synthetic data is not enough** — the iterative error-analysis-and-repair loop adapts the training distribution to the specific failure modes of the downstream classifier, producing far better data than a single-pass generation. The paper shows this approach outperforms GLiClass (151M params) on 3/4 benchmarks despite being 27% smaller, and in the zero-shot regime beats SetFit trained with 8 real labeled examples.

## Step-by-Step Workflow

1. **Define the classification task.** Collect the set of class labels, optional natural-language descriptions for each class, and any domain context. If the user has a few real examples (k=2, 4, or 8 per class), include them as seeds — the framework works with zero examples but benefits from seeds.

2. **Generate balanced seed training data.** Prompt the LLM to produce N synthetic examples per class (start with 16-32 per class). Enforce constraints: class balance, no duplicate or near-duplicate texts, diverse linguistic patterns, and semantic alignment with label descriptions. Store as a labeled dataset.

3. **Generate a synthetic validation set.** Separately prompt the LLM to produce a held-out validation set (8-16 examples per class). Ensure no overlap with training examples via deduplication. This set remains fixed across iterations to measure genuine improvement.

4. **Train the lightweight classifier.** Use SetFit (or another contrastive/fine-tuning approach) on the synthetic training set. SetFit with `sentence-transformers/all-mpnet-base-v2` is the recommended backbone. Train with standard contrastive learning followed by a classification head.

5. **Evaluate and produce diagnostics.** Run the trained classifier on the validation set. Compute: overall accuracy, macro-F1, per-class precision/recall/F1, a full confusion matrix, and example-level error traces (the actual text, predicted label, and true label for every misclassified example).

6. **Analyze failure modes.** Feed the diagnostics to the LLM with a structured prompt: "Given these evaluation results, identify (a) which class pairs are most confused, (b) what linguistic patterns cause errors (negation, ambiguity, lexical overlap), (c) which classes have lowest recall and why." The LLM produces a concise error analysis summary.

7. **Synthesize targeted repair examples.** Based on the error analysis, prompt the LLM to generate targeted examples: hard negatives for the most-confused class pairs, boundary cases that disambiguate similar classes, paraphrases that cover missing linguistic patterns, and domain-specific variations for low-recall classes. Add these to the training set.

8. **Check convergence.** If validation macro-F1 improved by more than epsilon (e.g., 0.5%) over the previous best, continue to step 4. If no improvement for p consecutive iterations (e.g., p=2), or if the maximum iteration count T_max is reached (e.g., T_max=5), stop.

9. **Export the final classifier.** Save the trained SetFit model, the final synthetic dataset, the validation set, and the iteration-by-iteration performance log. The model is ready for deployment at sentence-transformer inference speeds.

10. **Validate on real data if available.** If the user has any held-out real labeled data, run the final classifier against it as a sanity check. Report results alongside the synthetic validation metrics.

## Concrete Examples

**Example 1: Sentiment classifier with no labeled data**

User: "I need a sentiment classifier for product reviews with 5 classes (very negative, negative, neutral, positive, very positive) but I have no labeled data."

Approach:
1. Define 5 sentiment classes with descriptions: "very negative — expresses strong dissatisfaction, anger, or frustration", etc.
2. Generate 32 synthetic product reviews per class (160 total), ensuring variety in product domains (electronics, clothing, food, services).
3. Generate a separate validation set of 10 per class (50 total).
4. Train SetFit on the 160 examples with contrastive learning.
5. Evaluate: confusion matrix reveals "neutral" and "negative" are frequently confused (precision 0.60 for neutral).
6. Error analysis: the LLM identifies that mild complaints ("it's okay but could be better") are labeled neutral by the classifier but are actually negative, and factual descriptions ("the battery is 3000mAh") are misclassified as positive.
7. Generate 20 targeted examples: 10 mild-complaint negatives and 10 neutral factual descriptions.
8. Retrain — neutral precision jumps from 0.60 to 0.78. Continue one more iteration targeting remaining edge cases.
9. Final model achieves macro-F1 of 0.74 on synthetic validation, comparable to zero-shot GPT performance.

Output:
```
Iteration 1: macro-F1 = 0.65 | Top confusion: neutral<->negative (38% of errors)
Iteration 2: macro-F1 = 0.74 | Top confusion: neutral<->positive (22% of errors)
Iteration 3: macro-F1 = 0.76 | Converged (delta < 0.5%)
Final model: SetFit (110M params) — 0.76 macro-F1, ~2ms/inference
```

**Example 2: Intent classification for a chatbot**

User: "Build me an intent classifier for a banking chatbot. Intents: check_balance, transfer_money, report_fraud, open_account, general_inquiry. I have 3 real examples per intent."

Approach:
1. Use the 15 real examples (3 per class) as seeds alongside label descriptions.
2. Generate 24 additional synthetic examples per class (120 total), conditioned on the seeds to match the domain register ("Hi, I'd like to check my savings balance" vs. academic style).
3. Generate validation set of 12 per class (60 total).
4. Train SetFit, evaluate. Error trace shows `transfer_money` confused with `check_balance` when users say things like "How much can I send?" (ambiguous reference to balance).
5. Generate 15 hard negatives: transfer requests that mention amounts, balance checks that mention other accounts.
6. Retrain. Next iteration reveals `general_inquiry` is a catch-all absorbing edge cases from other intents.
7. Generate 10 boundary examples for `general_inquiry` that are clearly general ("What are your hours?") and 10 near-miss examples that belong to specific intents but sound general.
8. Converge after 3 iterations at macro-F1 = 0.89.

Output:
```
Iteration 1: macro-F1 = 0.78 | Confused: transfer_money<->check_balance
Iteration 2: macro-F1 = 0.85 | Confused: general_inquiry absorbing other intents
Iteration 3: macro-F1 = 0.89 | Converged
Exported: setfit-banking-intent/ (110M params, <3ms inference)
Synthetic dataset: 195 training examples, 60 validation examples
```

**Example 3: Topic classification for news articles**

User: "Classify news articles into World, Sports, Business, Sci/Tech. Zero labeled examples. I need something I can deploy cheaply."

Approach:
1. Define 4 classes with descriptions grounded in news domains.
2. Generate 32 diverse synthetic headlines + lead paragraphs per class (128 total). Include variety in regions, sports, industries, and tech subfields.
3. Validation set: 16 per class (64 total).
4. Train SetFit. First evaluation: macro-F1 = 0.82. Main confusion: Business<->Sci/Tech for fintech and tech-company earnings articles.
5. Error analysis: LLM notes that articles about tech companies' financial performance get split between Business and Sci/Tech depending on whether the focus is the stock price or the product.
6. Generate 20 targeted examples: 10 fintech articles clearly in Business (focus on revenue, stock, M&A) and 10 clearly in Sci/Tech (focus on product launches, research breakthroughs at tech companies).
7. Retrain. Macro-F1 reaches 0.88. Next iteration shows minimal gain — converge.

Output:
```
Iteration 1: macro-F1 = 0.82 | Confused: Business<->Sci/Tech (fintech overlap)
Iteration 2: macro-F1 = 0.88 | Converged
Model size: 110M params | Inference: ~2ms/example on CPU
```

## Best Practices

- **Do:** Always generate the validation set separately from training data and keep it fixed across iterations. This prevents overfitting the loop to self-confirming data.
- **Do:** Enforce strict deduplication (exact and near-duplicate via cosine similarity > 0.95) across all generated examples. Duplicates waste budget and inflate metrics.
- **Do:** Include domain context and register in generation prompts. "Generate a product review" produces different text than "Generate a formal academic abstract" — match the target distribution.
- **Do:** Cap per-class examples (e.g., 64-128 total after all iterations) to prevent training set bloat that slows iteration without helping.
- **Avoid:** Generating all synthetic data in one massive batch. The iterative refinement is the core contribution — single-pass generation produces generic data that misses classifier-specific failure modes.
- **Avoid:** Using the LLM's own zero-shot predictions as validation labels. The validation set labels come from the generation prompt's class assignments, not from re-classifying with the LLM.
- **Avoid:** Running too many iterations (>5-6). Diminishing returns set in quickly; most gains come in iterations 1-3.

## Error Handling

- **Synthetic data is too homogeneous:** If the classifier overfits quickly (training accuracy 99%, validation accuracy <60%), the generated examples lack diversity. Re-prompt with explicit diversity constraints: vary sentence length, vocabulary register, narrative perspective, and domain.
- **Validation set is not representative:** If the model performs well on synthetic validation but poorly on real data, the LLM's conception of the classes diverges from reality. Inject any available real examples as seeds, or refine class descriptions to be more concrete.
- **Class imbalance in errors:** If one class dominates the confusion matrix, check whether its label description is too broad or overlaps with another class. Sharpen descriptions before generating more data.
- **Loop does not converge:** If macro-F1 oscillates without improving after 3+ iterations, the task may be too ambiguous for the label set. Consider merging confusable classes or adding a class to capture the ambiguous boundary.
- **SetFit training instability:** With very small datasets (<8 examples per class), contrastive learning can be unstable. Use more contrastive pairs per example (increase `num_iterations` in SetFit config) or generate more seed examples in the first Curate phase.

## Limitations

- **Label quality is bounded by LLM understanding.** If the LLM has a flawed or incomplete understanding of a class (e.g., domain-specific jargon, rare categories), the synthetic data will reflect that misunderstanding. Providing detailed class descriptions and seed examples mitigates this.
- **Not suited for tasks requiring world knowledge or reasoning.** The lightweight classifier only learns surface patterns from text. Tasks like fact-checking or temporal reasoning cannot be distilled this way.
- **Synthetic validation is an imperfect proxy.** The loop optimizes against LLM-generated validation data, which may not perfectly match real-world distributions. Always validate on real data when possible.
- **Multilingual performance is weaker.** The paper notes that EuroBERT (multilingual encoder) underperforms SetFit with English-centric `all-mpnet-base-v2`. For non-English tasks, choose an appropriate multilingual sentence transformer and expect to need more iterations.
- **Computational cost of iterations.** Each loop iteration requires LLM calls for generation and analysis plus model training. For very large label sets (50+ classes), budget and latency for the curation loop itself become non-trivial.

## Reference

**Paper:** Maheshwari, G. & El Haddad, K. (2026). *Curate-Train-Refine: A Closed-Loop Agentic Framework for Zero Shot Classification.* arXiv:2601.16530v1. [https://arxiv.org/abs/2601.16530v1](https://arxiv.org/abs/2601.16530v1)

Look for: Section 3 (Method) for the full agentic loop specification, Section 4 (Experiments) for benchmark results on SST-5/Emotion/CR/AG News, and the ablation study showing that iterative refinement outperforms single-pass generation by 5-12% macro-F1.