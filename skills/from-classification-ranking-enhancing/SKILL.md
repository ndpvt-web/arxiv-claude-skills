---
name: "from-classification-ranking-enhancing"
description: "Reframe subjective classification tasks as ranking problems with GRPO reinforcement learning. Use when building personality detection, sentiment ranking, subjective labeling, or any classification task where categories have blurred boundaries. Triggers: 'rank personality traits', 'MBTI detection', 'classification to ranking', 'subjective classification with LLM', 'ranking reward function', 'GRPO personality'"
---

# Classification-to-Ranking with GRPO for Subjective Trait Detection

This skill teaches Claude to apply the PerDet-R1 technique from "From Classification to Ranking" (AAAI 2026). The core idea: when classifying subjective attributes (personality, sentiment, style), reframe the problem as a **ranking task** where the model outputs an ordered list of candidate labels scored by relevance, then apply Group Relative Policy Optimization (GRPO) with a ranking-aware reward function (NDCG + dimension similarity). This consistently outperforms direct classification on tasks where category boundaries are fuzzy and partial correctness matters.

## When to Use

- When the user wants to detect MBTI personality types from social media posts or text
- When building a classifier for subjective categories where items can partially match multiple labels (e.g., personality, emotion, writing style)
- When the user asks to fine-tune an LLM for a classification task and the classes have blurred or overlapping boundaries
- When designing reward functions for GRPO/RLHF that need to handle ranked outputs rather than binary correct/incorrect
- When the user needs to move beyond prompt-only personality or trait detection toward a trainable pipeline
- When converting an existing binary classification system to a ranking-based system for better nuance

## Key Technique

**The Problem with Classification for Subjective Traits.** Standard approaches treat personality detection as four independent binary classifications (E/I, S/N, T/F, J/P), losing inter-trait relationships. An INTJ misclassified as INTP shares 3 of 4 dimensions — but binary classification treats this the same as getting all four wrong. Prompt-based methods depend entirely on expert-crafted knowledge with no autonomous pattern learning.

**Ranking Reformulation.** Instead of predicting one label, the model outputs a Top-K ranked list of all 16 MBTI types ordered by relevance to the input text. This captures the reality that a person's posts may show strong signals for some dimensions and ambiguous signals for others. The pipeline has two stages: (1) SFT on teacher-model-generated reasoning traces to establish ranking ability and enforce a structured output format (`<<think>>...<<answer>>[type1, type2, ..., typeK]`), and (2) GRPO reinforcement learning with a ranking-aware reward.

**The Ranking Reward Function.** The reward combines two signals. First, NDCG@K (Normalized Discounted Cumulative Gain) scores the full ranked list — types closer to the ground truth appearing higher in the ranking earn more reward, with logarithmic position discounting. Second, a Dimension Similarity (DS) score for the top-1 prediction counts matching MBTI dimensions with a small positional weight (epsilon=0.1), so getting 3/4 dimensions right earns partial credit. This prevents reward hacking and teaches the model to produce calibrated rankings even when the "correct" answer is debatable.

## Step-by-Step Workflow

1. **Define the label space and dimension structure.** Enumerate all target labels (e.g., 16 MBTI types) and identify the constituent dimensions (E/I, S/N, T/F, J/P). For non-MBTI tasks, define analogous axes — e.g., for emotion detection: valence, arousal, dominance.

2. **Prepare input data with truncation strategy.** For each user/document, collect text samples. Truncate each post to a fixed token limit (128 tokens) and cap the number of posts per user (50 max). Concatenate with separators to form the input context.

3. **Generate reasoning traces with a teacher model.** Use a strong LLM (e.g., Qwen-plus, GPT-4) to produce chain-of-thought reasoning for each sample. The teacher model should analyze the text across each dimension and output a ranked list of Top-K candidate types. Enforce the output format:
   ```
   <<think>> Analysis of traits across dimensions... <</think>>
   <<answer>>[INTJ, INFJ, INTP]<</answer>>
   ```

4. **Apply rejection sampling to build SFT data.** Filter teacher outputs to keep only samples where the ground-truth label appears in the Top-K predictions. This yields high-quality training pairs. Target approximately 1,000 curated samples for SFT.

5. **Run supervised fine-tuning (SFT).** Fine-tune the base model (e.g., Qwen2.5-7B-Instruct) on the curated data using LoRA (rank 32). Train for 3 epochs with batch size 256. This stage teaches the model the ranking output format and basic trait-analysis reasoning.

6. **Implement the ranking reward function.** Code the reward as a combination of:
   - **NDCG@K**: `DCG@k = sum((2^s_i - 1) / log2(i+1))` where `s_i` is the dimension similarity between the i-th predicted type and the ground truth. Normalize by IDCG (ideal ordering).
   - **Dimension Similarity**: `DS(pred, truth) = sum((1 + epsilon*j) * match(pred_dim_j, truth_dim_j))` for each of the 4 dimensions, with epsilon=0.1.

7. **Run GRPO reinforcement learning.** Sample groups of 16 completions per prompt at temperature 1.0. Compute rewards for each, calculate group-relative advantages `A_i = (r_i - mean) / std`, and optimize the clipped policy objective with KL regularization against the SFT checkpoint. Train for 2,000 steps with batch size 128.

8. **Extract final predictions from ranked output.** At inference, take the top-1 type from the ranked list as the predicted label. Optionally expose the full ranking to users for transparency and confidence assessment.

9. **Evaluate with both binary and multi-class metrics.** Report per-dimension binary Macro-F1 (averaged over E/I, S/N, T/F, J/P) and 16-class F1 for the full MBTI type. The ranking approach disproportionately improves multi-class F1 since it captures cross-dimension patterns.

10. **Iterate on reward function weights.** If the model collapses to always predicting common types, increase the NDCG weight relative to DS. If it gets individual dimensions right but ranks poorly, increase epsilon to weight later dimensions more.

## Concrete Examples

**Example 1: Building an MBTI detector from Reddit posts**

User: "I have a dataset of Reddit users with their posts and MBTI labels. I want to build a personality detector that's better than simple classification."

Approach:
1. Load the dataset and truncate each post to 128 tokens, keeping up to 50 posts per user
2. Format input as: `"Based on the following social media posts, rank the most likely MBTI types for this user:\n\nPost 1: {text}\nPost 2: {text}\n..."`
3. Use a teacher model to generate reasoning traces with Top-3 ranked MBTI predictions
4. Filter with rejection sampling — keep only samples where ground truth is in Top-3
5. SFT on ~1,000 filtered samples with structured output format
6. Implement NDCG@3 + DS reward function
7. Run GRPO for 2,000 steps

Output format from the trained model:
```
<<think>>
The user's posts show strong introspective tendencies ("I spent the weekend
analyzing..."), preference for abstract concepts over concrete details
("the pattern behind..."), logical argumentation style, and flexible
scheduling language ("whenever I get around to it"). This suggests
Introversion, iNtuition, Thinking, and Perceiving as primary traits.
<</think>>
<<answer>>[INTP, INTJ, ENTP]<</answer>>
```

**Example 2: Adapting the technique to emotion intensity ranking**

User: "I want to classify customer reviews into emotion categories, but some reviews express mixed emotions. Can I use ranking instead?"

Approach:
1. Define label space: [joy, anger, sadness, fear, surprise, disgust, neutral] and identify constituent dimensions (valence: positive/negative, arousal: high/low, target: self/other)
2. Reframe as ranking: model outputs emotions ordered by intensity in the review
3. Generate teacher traces that reason about which emotions are primary vs secondary
4. Build dimension similarity score: count matching valence, arousal, and target dimensions
5. NDCG@3 reward scores the full ranking; DS reward scores the top-1 prediction
6. SFT then GRPO as in the MBTI pipeline

Output:
```
<<think>>
"I was so disappointed with the product but honestly I'm more angry at
myself for not reading reviews first." Primary emotion is anger (self-directed,
high arousal, negative valence). Secondary is sadness (low arousal, negative
valence, self-directed).
<</think>>
<<answer>>[anger, sadness, disgust]<</answer>>
```

**Example 3: Implementing the NDCG reward function**

User: "How do I code the ranking reward for GRPO?"

```python
import numpy as np

def dimension_similarity(predicted_type: str, ground_truth: str, epsilon: float = 0.1) -> float:
    """Score how many MBTI dimensions match between predicted and ground truth."""
    # Map MBTI type to 4 binary dimensions
    dims_pred = list(predicted_type)   # e.g., ['I', 'N', 'T', 'P']
    dims_true = list(ground_truth)     # e.g., ['I', 'N', 'T', 'J']
    score = 0.0
    for i in range(4):
        if dims_pred[i] == dims_true[i]:
            score += (1 + epsilon * i)
    return score

def ndcg_reward(ranked_predictions: list[str], ground_truth: str, k: int = 3, epsilon: float = 0.1) -> float:
    """Compute NDCG@k using dimension similarity as relevance scores."""
    # Compute DCG
    dcg = 0.0
    for i, pred in enumerate(ranked_predictions[:k]):
        relevance = dimension_similarity(pred, ground_truth, epsilon)
        dcg += (2 ** relevance - 1) / np.log2(i + 2)  # i+2 because rank starts at 1

    # Compute IDCG (ideal: ground truth first, then closest types)
    all_types = generate_all_mbti_types()  # 16 types
    all_scores = sorted(
        [dimension_similarity(t, ground_truth, epsilon) for t in all_types],
        reverse=True
    )
    idcg = sum((2 ** s - 1) / np.log2(i + 2) for i, s in enumerate(all_scores[:k]))

    return dcg / idcg if idcg > 0 else 0.0

def combined_reward(ranked_predictions: list[str], ground_truth: str) -> float:
    """Full reward: NDCG for ranking quality + DS for top-1 accuracy."""
    ndcg = ndcg_reward(ranked_predictions, ground_truth, k=3)
    ds = dimension_similarity(ranked_predictions[0], ground_truth)
    return ndcg + ds
```

## Best Practices

- **Do:** Use rejection sampling during SFT data construction. Only keep teacher outputs where the ground truth appears in the Top-K — this dramatically improves SFT quality over using all samples.
- **Do:** Enforce a strict output format with delimiters (`<<think>>`, `<<answer>>`) during SFT. The GRPO stage is unstable without a reliable parsing format.
- **Do:** Set K=3 for Top-K ranking. K=1 collapses to classification; K>5 dilutes the reward signal with low-relevance types.
- **Do:** Use the dimension similarity score with positional weighting (epsilon=0.1) rather than exact-match-only. This gives partial credit that stabilizes GRPO training.
- **Avoid:** Training GRPO without the SFT initialization. The model needs to produce parseable ranked outputs before reward optimization can work.
- **Avoid:** Using binary accuracy as the sole evaluation metric. The ranking approach shines on multi-class F1 and calibration metrics — report both binary per-dimension F1 and full 16-class F1.
- **Avoid:** Treating all MBTI dimensions as equally easy to predict. T/F and J/P typically have lower inter-annotator agreement; consider dimension-specific analysis.

## Error Handling

| Problem | Cause | Fix |
|---------|-------|-----|
| Model outputs unparseable format | SFT training insufficient or format not enforced | Increase SFT epochs; add format-compliance reward term in GRPO |
| GRPO reward collapses to constant | All generations in a group produce identical rankings | Increase sampling temperature; increase group size from 16 to 32 |
| Top-1 accuracy drops during GRPO | Reward over-optimizes ranking at expense of top position | Increase weight of DS reward relative to NDCG |
| Model always predicts dominant type (e.g., INFP) | Class imbalance in training data | Add class-balancing during SFT data sampling; check reward isn't biased |
| Reasoning traces are generic | Teacher model didn't produce trait-specific analysis | Improve teacher prompt with explicit per-dimension analysis instructions |

## Limitations

- **Requires labeled training data.** This is a fine-tuning approach, not zero-shot. You need ground-truth personality labels to compute rewards.
- **MBTI validity concerns.** MBTI itself is debated in psychology. The technique is sound for any categorical-with-dimensions labeling scheme, but MBTI-specific results should be interpreted cautiously.
- **Compute cost.** GRPO with 16 generations per prompt at 2,000 steps requires significant GPU time. Not suitable for quick prototyping — start with SFT-only and add GRPO when SFT plateaus.
- **Language and cultural bias.** Models trained on English forum data (PersonalityCafe, Reddit) may not generalize to other languages or cultural contexts where personality expression differs.
- **Not applicable to objective classification.** If categories have clear, unambiguous boundaries (spam/not-spam, language identification), standard classification is simpler and sufficient. The ranking approach specifically helps when categories overlap.

## Reference

**Paper:** [From Classification to Ranking: Enhancing LLM Reasoning Capabilities for MBTI Personality Detection](https://arxiv.org/abs/2601.18582v1) (AAAI 2026 Bridge). Look for: Section 3 (methodology) for the full SFT+GRPO pipeline, Equation 5-8 for the NDCG and DS reward formulas, and Table 1-2 for benchmark results showing 2.78% binary F1 and 8.79% multi-class F1 improvements over previous SOTA.