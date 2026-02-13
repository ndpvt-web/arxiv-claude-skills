---
name: "beyond-holistic-scores-automatic"
description: "Build trait-based essay scoring systems that evaluate argumentative writing across multiple rubric dimensions (Content, Organization, Word Choice, Sentence Fluency, Conventions) using structured in-context learning prompts and ordinal regression. Trigger phrases: 'score essays by trait', 'build essay grading rubric', 'argumentative essay evaluation', 'trait-based writing assessment', 'rubric-aligned essay scoring', 'automated writing feedback'."
---

# Trait-Based Argumentative Essay Scoring

This skill enables Claude to build automated essay scoring systems that go beyond single holistic scores by evaluating argumentative writing across five distinct quality traits: Ideas & Content, Organization, Word Choice, Sentence Fluency, and Conventions. The approach uses two complementary techniques from the paper: (1) structured in-context learning prompts with rubric-aligned exemplars for LLM-based scoring, and (2) a CORAL ordinal regression formulation that explicitly models the ordered nature of rubric scores, substantially outperforming standard classification and regression baselines.

## When to Use

- When the user asks to build an automated essay scoring or grading system
- When designing rubric-based evaluation prompts for any structured writing task (not just essays -- reports, proposals, code reviews)
- When the user needs to score text on an ordinal scale (e.g., 1-5 stars, weak/fair/strong) and wants to preserve score ordering in the model
- When building educational tools that provide per-dimension feedback on student writing
- When the user wants to evaluate LLM outputs against a quality rubric with multiple criteria
- When implementing ordinal regression with transformer encoders for any ranked classification task
- When the user asks about CORAL loss, ordinal classification, or rank-consistent training

## Key Technique

**Structured In-Context Learning for Trait Scoring.** Rather than asking an LLM to produce a single essay score, this approach prompts the model as an expert evaluator with a carefully structured prompt: (1) a role specification ("expert evaluator of students' argumentative essays"), (2) a trait definition pulled directly from the rubric, (3) rubric guidelines with one exemplar essay per score level, (4) the student essay, and (5) an output specification requesting JSON with justification, score, and confidence. Critically, the output specification must appear at the very end of the prompt -- placing it elsewhere causes models to ignore formatting instructions. The paper found that a single exemplar per score level works best; adding more exemplars degrades output quality for non-reasoning models.

**CORAL Ordinal Regression.** For supervised models, standard cross-entropy classification treats score levels as independent categories, discarding the fact that a score of 2 is closer to 3 than to 1. The CORAL (Consistent Ordinal Regression) framework fixes this by converting a K-level ordinal task into K-1 binary threshold predictions. For a 3-level scale (weak/fair/strong), the model predicts P(score > 1) and P(score > 2) using shared features but separate thresholds. Training uses binary cross-entropy on each threshold with per-threshold class weights to handle imbalance. At inference, threshold cutoffs (c1, c2) are grid-searched on a validation set with the monotonicity constraint c2 >= c1. This approach achieved QWK scores of 0.59 (Content), 0.53 (Organization), 0.61 (Word Choice), 0.47 (Sentence Fluency), and 0.48 (Conventions) on the ASAP++ dataset -- consistently outperforming both LLMs and standard classification/regression baselines.

**Score Remapping for Pedagogical Clarity.** The original 6-point essay scores are remapped to 3 levels (1-2 -> weak, 3-4 -> fair, 5-6 -> strong). This coarser scale improves inter-rater agreement, is more pedagogically actionable, and aligns better with how teachers actually use rubrics. When building scoring systems, collapsing fine-grained scales to meaningful tiers often improves both model performance and practical utility.

## Step-by-Step Workflow

### Path A: LLM-Based Scoring with Structured Prompts

1. **Define the trait rubric.** For each quality dimension, write a concise definition and describe what each score level looks like. Use established frameworks: Content (idea development and clarity), Organization (structure and coherence), Word Choice (vocabulary precision), Sentence Fluency (rhythm and variety), Conventions (grammar, spelling, punctuation).

2. **Select one exemplar per score level.** For each trait, choose one representative text per score tier (e.g., weak/fair/strong). These exemplars must not overlap with the texts you will score. Keep the total prompt under the model's context window -- one exemplar per level is optimal.

3. **Construct the structured prompt.** Assemble the prompt in this exact order: (a) Role: "You are an expert evaluator of [text type]"; (b) Context: genre, audience, expected length; (c) Trait definition from rubric; (d) Score level descriptions with one exemplar each; (e) The text to evaluate, clearly delimited; (f) Output specification requesting JSON with `reasoning`, `score`, and `confidence` fields. The output specification MUST be the final section.

4. **Score each trait independently.** Run the prompt once per trait. Do not ask the model to score all five traits in a single pass -- this reduces accuracy and makes it harder to debug.

5. **Parse and validate outputs.** Extract the JSON response. Flag any scores where confidence < 0.6 for human review. Confidence values are not calibrated probabilities but serve as useful triage signals.

6. **Aggregate and report.** Present per-trait scores with the model's justification text. Optionally compute a weighted holistic score from the trait scores.

### Path B: Supervised CORAL Ordinal Regression

1. **Prepare the dataset.** Collect texts with human-assigned ordinal scores per trait. If scores are on a fine-grained scale (e.g., 1-6), remap to 3 pedagogically meaningful tiers. Split into train (60%), validation (20%), test (20%).

2. **Encode with a long-sequence transformer.** Use BigBird, Longformer, or any encoder supporting your max document length. Extract the [CLS] token representation from the final hidden layer.

3. **Build the CORAL head.** Replace the standard classification head with K-1 binary sigmoid outputs (where K is the number of ordinal levels). Each output predicts P(score > threshold_k). Apply dropout before the linear projection.

4. **Train with weighted binary cross-entropy.** For each threshold, compute positive-class weight as (num_negative / num_positive) from training data. Minimize the sum of K-1 binary cross-entropy losses. Train for up to 6 epochs with early stopping (patience=2) on validation QWK.

5. **Optimize decision thresholds.** After training, grid-search cutoff pairs (c1, c2, ..., c_{K-1}) on the validation set to maximize QWK, enforcing the monotonicity constraint c_{k+1} >= c_k.

6. **Evaluate with QWK.** Report Quadratic Weighted Kappa per trait on the test set. Run with 5+ random seeds and report mean +/- std.

## Concrete Examples

**Example 1: Building an LLM-based essay scorer**

User: "I want to score student argumentative essays on Content and Organization using an LLM."

Approach:
1. Define the two traits with rubric descriptions
2. Select exemplar essays for each score level
3. Build structured prompts for each trait

Output -- Content scoring prompt:
```json
{
  "role": "You are an expert evaluator of students' argumentative essays written by 13-year-old students.",
  "trait": "Ideas and Content: How fully the essay develops its central argument, how clearly ideas are expressed, and how well evidence supports the thesis.",
  "rubric": {
    "weak (1)": "Ideas are unclear or undeveloped. Little or no evidence supports the argument. Example: [exemplar essay text]",
    "fair (2)": "Ideas are present but inconsistently developed. Some evidence is provided but may be irrelevant. Example: [exemplar essay text]",
    "strong (3)": "Ideas are clearly articulated and well-developed. Evidence is relevant and effectively supports the argument. Example: [exemplar essay text]"
  },
  "essay": "[student essay text here]",
  "output_format": "Respond in JSON with keys: reasoning (2-3 sentences justifying your score), score (1, 2, or 3), confidence (0.0 to 1.0)"
}
```

Expected model output:
```json
{
  "reasoning": "The essay presents a clear thesis about school uniforms but develops only two supporting points, one of which relies on personal anecdote rather than evidence. The counterargument is acknowledged but not fully addressed.",
  "score": 2,
  "confidence": 0.75
}
```

**Example 2: Implementing CORAL ordinal regression in PyTorch**

User: "I have essay scores on a 1-3 scale. Help me implement CORAL loss for a BigBird encoder."

Approach:
1. Build the CORAL head on top of the encoder
2. Implement the loss function
3. Add threshold optimization at inference

```python
import torch
import torch.nn as nn

class CORALHead(nn.Module):
    """CORAL ordinal regression head for K ordinal classes."""

    def __init__(self, hidden_size: int, num_classes: int, dropout: float = 0.1):
        super().__init__()
        self.num_thresholds = num_classes - 1  # K-1 binary tasks
        self.dropout = nn.Dropout(dropout)
        # Shared feature projection, separate bias per threshold
        self.fc = nn.Linear(hidden_size, 1, bias=False)
        self.thresholds = nn.Parameter(torch.zeros(self.num_thresholds))

    def forward(self, cls_hidden_state: torch.Tensor) -> torch.Tensor:
        """Returns logits of shape (batch, num_thresholds)."""
        x = self.dropout(cls_hidden_state)
        logits = self.fc(x) + self.thresholds  # (batch, num_thresholds)
        return logits


def coral_loss(logits: torch.Tensor, labels: torch.Tensor,
               threshold_weights: list[torch.Tensor]) -> torch.Tensor:
    """
    Binary cross-entropy CORAL loss.
    logits: (batch, K-1) raw logits per threshold
    labels: (batch,) integer labels in [0, K-1]
    threshold_weights: list of K-1 positive-class weight scalars
    """
    num_thresholds = logits.size(1)
    total_loss = 0.0
    for k in range(num_thresholds):
        binary_target = (labels > k).float()
        weight = threshold_weights[k]
        pos_weight = torch.tensor([weight], device=logits.device)
        total_loss += nn.functional.binary_cross_entropy_with_logits(
            logits[:, k], binary_target, pos_weight=pos_weight
        )
    return total_loss / num_thresholds


def decode_coral(logits: torch.Tensor, cutoffs: list[float]) -> torch.Tensor:
    """Convert CORAL logits to ordinal class using optimized cutoffs."""
    probs = torch.sigmoid(logits)  # (batch, K-1)
    preds = torch.zeros(logits.size(0), dtype=torch.long, device=logits.device)
    for k, c in enumerate(cutoffs):
        preds += (probs[:, k] > c).long()
    return preds
```

**Example 3: Adapting trait scoring for code review quality**

User: "I want to evaluate code review comments on multiple quality dimensions."

Approach:
1. Define traits analogous to essay rubrics: Specificity (like Content), Structure (like Organization), Terminology (like Word Choice), Clarity (like Sentence Fluency), Correctness (like Conventions)
2. Build a 3-level rubric per trait
3. Use the structured prompt template with code review exemplars

```
Trait: Specificity
- Weak: Comment is vague ("this looks wrong") with no actionable guidance
- Fair: Comment identifies the issue but suggestion is incomplete or generic
- Strong: Comment pinpoints the exact problem, explains why, and suggests a concrete fix

Prompt structure:
  Role -> Trait definition -> Rubric with exemplar comments -> Target comment -> JSON output spec
```

## Best Practices

- **Do:** Place the output format specification at the very end of the prompt. Models frequently ignore formatting instructions placed earlier.
- **Do:** Use one exemplar per score level, not more. Additional exemplars cause non-reasoning models to produce malformed outputs.
- **Do:** Remap fine-grained scales (1-6, 1-10) to 3 meaningful tiers. This improves model agreement with human raters and produces more actionable feedback.
- **Do:** Compute per-threshold class weights for CORAL training to handle score distribution imbalance.
- **Do:** Score each trait in a separate LLM call rather than asking for all traits simultaneously.
- **Avoid:** Treating ordinal scores as nominal categories in supervised models. Standard cross-entropy discards the ordering information that is central to rubric-based scoring.
- **Avoid:** Interpreting LLM confidence values as calibrated probabilities. Use them only as triage flags (e.g., route low-confidence cases to human review).
- **Avoid:** Using temperature=0 for LLM scoring. The paper uses temperature=0.8 and averages across runs to get more stable estimates.

## Error Handling

- **Malformed JSON output:** Wrap LLM calls in a retry loop (max 3 attempts). If the model still fails, extract the score via regex fallback on the raw text.
- **All predictions collapse to one class:** Check per-threshold class weights in CORAL. Extreme imbalance (>10:1) may require oversampling the minority class or adjusting the weight computation.
- **QWK near zero or negative:** Verify label encoding matches the ordinal scale (0-indexed vs 1-indexed). Ensure the monotonicity constraint c2 >= c1 is enforced during threshold grid search.
- **Score distribution mismatch:** If the model systematically over- or under-scores a trait, recalibrate thresholds on a held-out calibration set. For LLM scoring, revise the exemplar for the under-represented level.
- **Context window overflow:** With in-context exemplars, long essays may exceed the context limit. Truncate essays to the first 4096 tokens (BigBird's limit) or summarize before scoring.

## Limitations

- The ASAP++ results are on English argumentative essays from 13-year-old students. Performance on other genres (narrative, expository), languages, or age groups is untested.
- LLM-based scoring (QWK ~0.49 for Content with Ministral-3) lags behind the supervised CORAL approach (QWK ~0.59) and both lag behind human inter-rater agreement on some traits.
- Sentence Fluency and Conventions are the hardest traits to score automatically (QWK 0.47-0.48 even for the best model), likely because they depend on subtle linguistic patterns.
- The 3-level collapsed scale is more reliable but less granular. Applications requiring fine-grained distinctions (e.g., percentile ranking) may need the full scale with proportionally more training data.
- CORAL requires labeled training data with human-assigned ordinal scores per trait, which is expensive to collect for new domains.

## Reference

**Paper:** [Beyond Holistic Scores: Automatic Trait-Based Quality Scoring of Argumentative Essays](https://arxiv.org/abs/2602.04604v1) (Favero et al., 2026). Focus on Section 3 for the CORAL formulation, Section 4 for prompt design, and Table 1 for per-trait QWK results across all methods.