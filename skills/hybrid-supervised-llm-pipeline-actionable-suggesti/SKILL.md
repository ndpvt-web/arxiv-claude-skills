---
name: "hybrid-supervised-llm-pipeline-actionable-suggesti"
description: "Build hybrid classifier-then-LLM pipelines to extract actionable suggestions from unstructured customer reviews. Use when the user says 'extract suggestions from reviews', 'mine actionable feedback', 'analyze customer complaints for improvements', 'build a suggestion extraction pipeline', 'classify and cluster review feedback', or 'summarize actionable insights from user reviews'."
---

# Hybrid Supervised-LLM Pipeline for Actionable Suggestion Mining

This skill enables Claude to design and implement two-stage pipelines that combine a high-recall supervised classifier (Stage 1) with an instruction-tuned LLM (Stage 2) to extract, categorize, cluster, and summarize actionable suggestions from unstructured customer reviews. The key architectural insight is that a cheap, fast classifier filters candidate sentences first — tuned aggressively for recall so no suggestion is permanently lost — then an LLM performs the expensive fine-grained extraction and reasoning on a much smaller candidate set.

## When to Use

- When the user wants to extract improvement suggestions from customer reviews (hotels, restaurants, products, SaaS, apps)
- When building a pipeline to process large volumes of unstructured feedback into categorized, actionable items
- When the user needs to cluster similar suggestions across thousands of reviews into a ranked summary for product/ops teams
- When designing a system where missing a suggestion is costlier than surfacing a false positive (asymmetric error costs)
- When the user asks to combine a fine-tuned classifier with LLM-based extraction for cost efficiency at scale
- When adapting a suggestion mining system from one domain (e.g., hospitality) to another (e.g., food delivery, e-commerce)

## Key Technique

**The Unrecoverable False Negative Problem.** In a cascaded pipeline, Stage 1 acts as a hard filter: any sentence it discards is permanently lost to downstream stages. A missed suggestion (false negative) can never be recovered by the LLM. This is why the paper trains the RoBERTa classifier with a precision-recall surrogate loss that explicitly trades precision for recall. The decision threshold is tuned post-training to maximize recall (target: 0.95+), accepting that ~30-40% of passed sentences may not contain suggestions. This "noisy but complete" candidate set is far more useful than a "clean but incomplete" one, because the LLM in Stage 2 can easily reject false positives but cannot hallucinate missed suggestions.

**Four-Stage LLM Reasoning.** Once the classifier passes candidate sentences, the LLM performs four sequential operations: (1) **Extraction** — isolate the exact suggestion span from surrounding opinion/narrative text; (2) **Categorization** — assign each suggestion to a predefined taxonomy (e.g., "cleanliness", "staff behavior", "food quality", "pricing"); (3) **Clustering** — group semantically similar suggestions across reviews; (4) **Summarization** — produce concise, business-ready action items per cluster. Each stage uses structured prompts with explicit output format constraints to maintain control.

**Why hybrid beats prompt-only.** Running all reviews through an LLM directly is expensive and brittle — the LLM must simultaneously detect suggestions AND extract them, causing both recall drops and hallucinated suggestions. The classifier pre-filter reduces LLM input volume by 60-80% while guaranteeing high suggestion coverage. It also makes the LLM's job easier: it only needs to extract and reason, not detect.

## Step-by-Step Workflow

1. **Segment reviews into sentences.** Split each review into individual sentences using a sentence tokenizer (e.g., spaCy, NLTK punkt). Each sentence becomes the unit of classification. Preserve the review ID and sentence index for traceability.

2. **Build or configure the high-recall binary classifier.** If training from scratch, fine-tune a RoBERTa-base (or distilled variant) on labeled data where sentences are tagged as "contains suggestion" or "does not contain suggestion". Use a surrogate loss that penalizes false negatives more heavily than false positives (e.g., weighted binary cross-entropy with FN weight 3-5x FP weight, or Fbeta loss with beta=2). If no labeled data exists, use few-shot prompting with an LLM to generate silver labels for bootstrapping.

3. **Tune the classification threshold for high recall.** After training, sweep the decision threshold from 0.1 to 0.9 and select the threshold that achieves recall >= 0.95 on a held-out validation set. Log the precision at this threshold — expect 0.4-0.6 precision, which is acceptable because the LLM handles precision in Stage 2.

4. **Run the classifier on all review sentences.** Pass every sentence through the classifier. Retain all sentences scoring above the tuned threshold. Tag each with its confidence score for optional downstream prioritization.

5. **Extract suggestion spans with the LLM.** For each candidate sentence (or batch of sentences from the same review), prompt the LLM to extract the exact actionable suggestion, stripping opinion, sentiment, and narrative. Use a structured output format (JSON) with fields: `original_sentence`, `extracted_suggestion`, `confidence`.

6. **Categorize each suggestion.** Prompt the LLM to assign each extracted suggestion to one or more categories from a predefined taxonomy. Provide the taxonomy in the system prompt. Output: `suggestion`, `category`, `subcategory`.

7. **Cluster similar suggestions.** Group suggestions with the same category, then prompt the LLM (or use embedding similarity + agglomerative clustering) to merge near-duplicate suggestions. Each cluster gets a representative label.

8. **Summarize clusters into action items.** For each cluster, prompt the LLM to generate a single concise action item with: the improvement needed, frequency/volume indicator (how many reviews mentioned it), and representative quotes.

9. **Rank and format the final output.** Sort action items by cluster size (frequency) descending. Format as a structured report with categories, action items, supporting evidence, and priority indicators.

10. **Validate with spot-checks.** Sample 20-30 original reviews and trace them through the pipeline to verify no obvious suggestions were dropped by the classifier and no hallucinated suggestions were introduced by the LLM.

## Concrete Examples

**Example 1: Hotel review analysis**

User: "I have 5,000 hotel reviews from TripAdvisor. Extract actionable suggestions the hotel management can act on."

Approach:
1. Split reviews into ~25,000 sentences
2. Classify each sentence — ~4,000 flagged as potential suggestions (high-recall filter)
3. LLM extracts suggestions from the 4,000 candidates, yielding ~2,200 genuine suggestions
4. Categorize into taxonomy: {Cleanliness, Staff, Amenities, Food, Location, Pricing, Room Quality, Noise, Check-in/out}
5. Cluster within each category
6. Summarize into ranked action items

Output:
```json
{
  "action_items": [
    {
      "rank": 1,
      "category": "Room Quality",
      "action": "Replace mattresses in rooms on floors 3-5; multiple guests report sagging and discomfort",
      "mention_count": 87,
      "representative_quotes": [
        "The mattress was so worn out I could feel the springs",
        "Bed desperately needs replacing, woke up with back pain"
      ]
    },
    {
      "rank": 2,
      "category": "Check-in/out",
      "action": "Add self-service kiosk or mobile check-in option to reduce wait times during peak hours",
      "mention_count": 64,
      "representative_quotes": [
        "Waited 40 minutes to check in, they should have an express option",
        "Would be great if they offered mobile check-in like other chains"
      ]
    }
  ]
}
```

**Example 2: Building the classifier with no labeled data**

User: "I want to build the suggestion classifier but I don't have labeled training data."

Approach:
1. Sample 500 review sentences from the target domain
2. Use Claude to label each sentence as suggestion/non-suggestion with explanations (silver labeling)
3. Manually verify a 50-sentence subset to calibrate LLM labeling quality
4. Fine-tune a distilroberta-base on the silver-labeled data with weighted BCE loss (FN_weight=4)
5. Evaluate on a manually labeled 100-sentence test set; tune threshold for recall >= 0.95
6. Deploy as the Stage 1 filter

Output (classifier training script skeleton):
```python
from transformers import AutoModelForSequenceClassification, AutoTokenizer, Trainer, TrainingArguments
import torch
import torch.nn as nn

class HighRecallLoss(nn.Module):
    def __init__(self, fn_weight=4.0):
        super().__init__()
        # Weight false negatives 4x more than false positives
        self.weight = torch.tensor([1.0, fn_weight])

    def forward(self, logits, labels):
        return nn.functional.cross_entropy(logits, labels, weight=self.weight.to(logits.device))

model = AutoModelForSequenceClassification.from_pretrained("distilroberta-base", num_labels=2)
tokenizer = AutoTokenizer.from_pretrained("distilroberta-base")

# After training, tune threshold:
# for threshold in [0.1, 0.15, 0.2, ..., 0.5]:
#     preds = (probs[:, 1] >= threshold).int()
#     recall = recall_score(labels, preds)
#     if recall >= 0.95:
#         selected_threshold = threshold; break
```

**Example 3: Prompt-only baseline (no classifier, LLM-only)**

User: "I only have 200 reviews. Is the classifier stage worth it or should I just use the LLM directly?"

Approach:
For small volumes (< 500 reviews), skip the classifier and run the full LLM pipeline directly. The classifier stage adds value at scale (thousands+ reviews) where LLM cost and latency matter.

Prompt for direct LLM extraction:
```
You are analyzing customer reviews to extract actionable suggestions.

For each review, extract ONLY explicit or strongly implied suggestions for improvement.
Do NOT extract complaints, opinions, or praise — only actionable directives.

For each suggestion found, output:
- extracted_suggestion: the specific improvement action
- category: one of [Cleanliness, Staff, Amenities, Food, Pricing, Room Quality, Noise, Check-in/out, Other]
- verbatim_quote: the exact text from the review supporting this suggestion

If a review contains no actionable suggestions, output: {"suggestions": []}

Review: "{review_text}"
```

Output:
```json
{
  "suggestions": [
    {
      "extracted_suggestion": "Add more power outlets near the desk area",
      "category": "Room Quality",
      "verbatim_quote": "there was only one outlet and it was behind the bed, they really need more plugs near the work desk"
    }
  ]
}
```

## Best Practices

- **Do:** Tune the classifier threshold on a validation set from the TARGET domain, not a generic benchmark. Recall requirements may vary: 0.95 for high-stakes domains (healthcare feedback), 0.90 for lower-stakes (product reviews).
- **Do:** Batch candidate sentences by review when prompting the LLM for extraction — co-located sentences from the same review provide context that improves extraction accuracy.
- **Do:** Define the category taxonomy BEFORE running the pipeline. An open-ended "categorize this" prompt produces inconsistent labels. Provide the taxonomy explicitly in the prompt.
- **Do:** Include a "no suggestion found" output option in the LLM extraction prompt so it can reject classifier false positives cleanly rather than hallucinating a suggestion.
- **Avoid:** Training the classifier to maximize F1 — balanced F1 optimizes for equal precision/recall, but this pipeline specifically needs asymmetric recall-heavy performance.
- **Avoid:** Using the LLM to both detect AND extract suggestions in a single pass at scale. This conflates two tasks and causes both recall drops (missed suggestions) and precision drops (hallucinated suggestions). Separate detection (classifier) from extraction (LLM).

## Error Handling

- **Classifier misses domain-specific suggestions:** If the classifier was trained on hospitality data but applied to SaaS reviews, recall may drop. Detect this by sampling 50 classifier-rejected sentences and checking for missed suggestions. If miss rate > 10%, retrain or lower the threshold further.
- **LLM hallucinates suggestions:** The LLM may fabricate suggestions not present in the source text. Mitigate by requiring verbatim quote evidence for each suggestion and programmatically verifying the quote exists in the source review (fuzzy string match, threshold > 0.8).
- **Inconsistent categorization:** If the LLM assigns different categories to obviously similar suggestions across batches, add 2-3 few-shot examples per category in the prompt, or run a second categorization pass on the full suggestion list for consistency.
- **Degenerate clusters:** If clustering produces one giant cluster and many singletons, adjust the similarity threshold or switch from LLM-based clustering to embedding-based (e.g., sentence-transformers + HDBSCAN) for more granular control.
- **Empty pipeline output:** If the classifier passes very few candidates, the threshold is too high or the training data doesn't match the target domain. Check threshold and domain alignment.

## Limitations

- The classifier requires labeled or silver-labeled data for the target domain. Zero-shot transfer across very different domains (e.g., hospitality to medical devices) degrades recall significantly.
- LLM-based clustering is less reproducible than embedding-based methods — running the same prompt twice may yield different cluster boundaries. For production systems, prefer embedding + deterministic clustering algorithms.
- The pipeline assumes reviews are in a single language. Multilingual reviews need per-language classifiers or a multilingual base model (XLM-RoBERTa).
- Implicit suggestions ("the wifi was painfully slow") are harder to detect than explicit ones ("they should upgrade their wifi"). The classifier captures explicit suggestions well but recall drops 10-15% on implicit suggestions.
- For fewer than ~500 reviews, the overhead of training and running a classifier is not justified — use the LLM-only approach instead.

## Reference

[A Hybrid Supervised-LLM Pipeline for Actionable Suggestion Mining in Unstructured Customer Reviews](https://arxiv.org/abs/2601.19214v1) — Trivedi et al., EACL 2026 Industry Track. Key sections: the precision-recall surrogate loss formulation (Section 3), threshold tuning protocol (Section 4), and the four-stage LLM prompt design for extraction/categorization/clustering/summarization (Section 5).