---
name: "robustexplain-evaluating-robustness-llm-based"
description: "Evaluate robustness of LLM-generated recommendation explanations under realistic user behavior noise. Use when: 'test explanation robustness', 'evaluate recommender explanation stability', 'perturb user history and measure explanation drift', 'benchmark LLM explanation consistency', 'build a robustness evaluation pipeline for recommendation agents', 'stress-test explanation generation under noisy inputs'."
---

# RobustExplain: Robustness Evaluation for LLM-Based Recommendation Explanations

This skill enables Claude to implement the RobustExplain evaluation framework — a systematic methodology for measuring how stable LLM-generated recommendation explanations are when user interaction histories are perturbed with realistic noise. The framework applies five perturbation types at graduated severity levels, then quantifies explanation drift across four complementary metrics (semantic, keyword, structural, length). Use this to build robustness benchmarks, identify fragile explanation behaviors, and harden recommendation explanation agents before deployment.

## When to Use

- When building or evaluating an LLM-based system that explains recommendations to users and you need to verify explanations don't collapse under noisy input histories
- When a user asks to stress-test or benchmark the consistency of generated explanations across perturbed inputs
- When designing a recommender system evaluation pipeline that goes beyond fluency/relevance to measure stability and trustworthiness
- When investigating how accidental clicks, missing metadata, temporal disorder, or preference drift affect explanation quality
- When comparing multiple LLMs (different sizes or families) on explanation robustness to select the most stable model
- When implementing a CI/CD quality gate that rejects explanation models whose robustness score falls below a threshold

## Key Technique

RobustExplain treats explanation robustness as a measurable property: given an explanation agent E, a user history Hu, and a perturbation function delta, robustness is `rho(E, delta) = E_over_Hu,r[sim(E(Hu, r, X), E(delta(Hu), r, X))]` — the expected similarity between explanations generated from the original and perturbed histories for the same recommended item r. This shifts evaluation from "is this explanation good?" to "does this explanation remain consistent when the input is realistically corrupted?"

The framework defines five perturbation types that model real-world noise sources: **Noise Injection** (random items simulating accidental clicks), **Temporal Shuffle** (randomizing interaction order), **Behavior Dilution** (injecting off-preference interactions from a shared device), **Category Drift** (replacing a fraction of interactions with alternative-category items), and **Missing Values** (dropping ratings, timestamps, or category metadata). Each perturbation has five severity levels (1=mild, 5=severe), parameterized so that level 5 corrupts roughly 50% of the signal.

Explanation similarity is measured across four orthogonal dimensions combined into a single score: **Semantic Similarity** (bag-of-words cosine similarity preserving overall meaning), **Keyword Stability** (Jaccard coefficient of extracted nouns/product names/categories), **Structural Consistency** (BLEU score capturing n-gram phrasing patterns), and **Length Stability** (relative length preservation). The combined robustness score is `rho = alpha1*Sem + alpha2*Key + alpha3*Struct + alpha4*Len` where weights sum to 1 (default: equal weighting at 0.25 each). Empirical baselines show current 7B-70B models achieve moderate robustness (~0.50), with structural consistency being the weakest dimension (0.378) and length the strongest (0.714).

## Step-by-Step Workflow

1. **Define the interaction history schema.** Structure user histories as ordered lists of interactions, each containing: item ID, item category, timestamp, rating (optional), and item features/metadata. Store as JSON or a pandas DataFrame with columns `[user_id, item_id, category, timestamp, rating, features]`.

2. **Build the explanation generation prompt template.** Use a structured prompt: `"Given the user's interaction history [HISTORY], explain why the item [ITEM] with features [FEATURES] is recommended. Focus on connecting the demonstrated preferences to item characteristics."` Format the history as a numbered list of past interactions with their metadata.

3. **Generate baseline explanations.** For each (user, recommended_item) pair in the evaluation set, call the LLM with the original unperturbed history to produce explanation `e`. Store these as the reference explanations.

4. **Implement the five perturbation functions**, each accepting a history and severity level s (1-5):
   - `noise_injection(H, s)`: Append `s` random items sampled uniformly from the item catalog.
   - `temporal_shuffle(H, s)`: Randomly permute `s/5` fraction of the interaction order.
   - `behavior_dilution(H, s)`: Insert `s` items from the user's least-engaged categories.
   - `category_drift(H, s)`: Replace `s/5` fraction of interactions with items from different categories.
   - `missing_values(H, s)`: Drop `s/5` fraction of metadata fields (ratings, timestamps, categories) at random.

5. **Generate perturbed explanations.** For each (user, item) pair, apply every perturbation at every severity level, then call the LLM with the perturbed history to produce explanation `e'`. This yields 25 perturbed explanations per pair (5 types x 5 levels).

6. **Compute the four robustness metrics** for each (e, e') pair:
   - `Sem(e, e')`: Tokenize both explanations, build TF vectors, compute cosine similarity.
   - `Key(e, e')`: Extract nouns, product names, categories, and adjectives via POS tagging; compute Jaccard similarity of keyword sets.
   - `Struct(e, e')`: Compute BLEU score (up to 4-grams) between e and e'.
   - `Len(e, e')`: Compute `1 - |len(e) - len(e')| / max(len(e), len(e'))`.

7. **Aggregate into combined robustness scores.** Compute `rho = 0.25*Sem + 0.25*Key + 0.25*Struct + 0.25*Len` per pair. Average across all pairs to get per-perturbation-type and per-severity-level scores. Report a grand average as the model's overall robustness score.

8. **Perform statistical validation.** Run paired t-tests between model variants or perturbation types. Report p-values and effect sizes (Cohen's d for pairwise, eta-squared for ANOVA across groups). Flag differences as significant only if p < 0.01.

9. **Visualize and interpret results.** Plot robustness scores as heatmaps (perturbation type x severity level), bar charts comparing models, and line charts showing degradation curves across severity. Identify the weakest perturbation type and metric dimension for each model.

10. **Set robustness thresholds and integrate into pipelines.** Establish a minimum acceptable robustness score (e.g., rho >= 0.55) as a deployment gate. Flag models or prompt configurations that fall below threshold on any single metric dimension.

## Concrete Examples

**Example 1: Evaluating a product recommendation explainer**

User: "I have an LLM that explains why products are recommended to users. I want to test how robust these explanations are to noisy user histories."

Approach:
1. Structure the user interaction data as JSON records with item_id, category, timestamp, rating, and features.
2. Generate reference explanations for 20 representative users across 5 recommended items each.
3. Apply all five perturbation functions at severity levels 1, 3, and 5 (reduced sweep for faster iteration).
4. Compute all four metrics and the combined score.

Output:
```
Robustness Report — GPT-based Explainer
========================================
Overall Robustness (rho):     0.513

By Perturbation Type:
  Noise Injection:            0.528
  Temporal Shuffle:           0.499
  Behavior Dilution:          0.522
  Category Drift:             0.485
  Missing Values:             0.529

By Metric Dimension:
  Semantic Similarity:        0.603
  Keyword Stability:          0.480
  Structural Consistency:     0.378
  Length Stability:            0.714

Severity Degradation (level 1 -> 5): -1.7%
Weakest dimension: Structural Consistency (0.378)
Weakest perturbation: Category Drift (0.485)

Recommendation: Structural consistency is significantly below
other metrics. Consider constraining output format or using
structured explanation templates to improve phrasing stability.
```

**Example 2: Comparing two candidate models for deployment**

User: "We're choosing between a 7B and a 14B model for our recommendation explanation service. Which is more robust?"

Approach:
1. Run the full RobustExplain pipeline on both models with identical user histories and perturbation configurations.
2. Compute per-model robustness scores across all perturbation types and severity levels.
3. Run paired t-tests on matched (user, item, perturbation, severity) pairs to determine statistical significance.

Output:
```
Model Comparison Report
========================
                    7B Model    14B Model   Delta    p-value
Overall rho:        0.492       0.524       +6.5%    <0.001
Semantic:           0.581       0.618       +6.4%    <0.001
Keyword:            0.459       0.498       +8.5%    <0.001
Structural:         0.355       0.396       +11.5%   <0.001
Length:              0.698       0.722       +3.4%    0.012

Cohen's d: 0.42 (medium effect)
Conclusion: The 14B model is significantly more robust across
all dimensions, with the largest gain in structural consistency.
The improvement justifies the additional compute cost if
explanation trustworthiness is a priority.
```

**Example 3: Building a robustness CI gate for a recommendation service**

User: "Add a robustness check to our CI pipeline that fails if explanation stability drops below acceptable levels."

Approach:
1. Create a test fixture with 10 representative users and their interaction histories plus 3 recommended items each.
2. Implement the perturbation functions and metrics as a Python test module.
3. Run perturbations at severity level 3 (moderate) across all five types.
4. Assert that the combined robustness score exceeds 0.50 and no individual metric falls below 0.35.

Output:
```python
# test_explanation_robustness.py
import pytest
from robustexplain import (
    generate_explanation, apply_perturbation,
    compute_semantic_sim, compute_keyword_stability,
    compute_structural_consistency, compute_length_stability
)

PERTURBATION_TYPES = [
    "noise_injection", "temporal_shuffle",
    "behavior_dilution", "category_drift", "missing_values"
]
SEVERITY = 3
MIN_COMBINED = 0.50
MIN_PER_METRIC = 0.35

@pytest.fixture
def eval_pairs():
    """Load test users and recommended items."""
    return load_test_fixture("fixtures/robustness_eval.json")

def test_explanation_robustness(eval_pairs):
    scores = {"sem": [], "key": [], "struct": [], "len": []}
    for user, item in eval_pairs:
        e = generate_explanation(user.history, item)
        for ptype in PERTURBATION_TYPES:
            h_perturbed = apply_perturbation(user.history, ptype, SEVERITY)
            e_prime = generate_explanation(h_perturbed, item)
            scores["sem"].append(compute_semantic_sim(e, e_prime))
            scores["key"].append(compute_keyword_stability(e, e_prime))
            scores["struct"].append(compute_structural_consistency(e, e_prime))
            scores["len"].append(compute_length_stability(e, e_prime))

    avg = {k: sum(v) / len(v) for k, v in scores.items()}
    combined = sum(avg.values()) / 4

    assert combined >= MIN_COMBINED, (
        f"Combined robustness {combined:.3f} below threshold {MIN_COMBINED}"
    )
    for metric, value in avg.items():
        assert value >= MIN_PER_METRIC, (
            f"{metric} robustness {value:.3f} below threshold {MIN_PER_METRIC}"
        )
```

## Best Practices

- **Do** run all five perturbation types — they test orthogonal failure modes. Noise injection and missing values tend to score similarly, but category drift and temporal shuffle reveal different weaknesses.
- **Do** use equal metric weights (0.25 each) as a default, then adjust only if your application prioritizes certain dimensions (e.g., increase structural weight if you render explanations in a fixed UI template).
- **Do** include severity level 1 and 5 at minimum to measure the degradation slope, which is often more informative than absolute scores.
- **Do** use paired evaluation — always compare explanations for the same (user, item) pair to control for variance in content difficulty.
- **Avoid** evaluating on fewer than 50 explanation pairs per perturbation type; statistical tests become unreliable below this sample size.
- **Avoid** using embedding-based semantic similarity (e.g., sentence-transformers) as the sole metric — the paper shows bag-of-words cosine and keyword Jaccard capture complementary signal that dense embeddings miss.
- **Avoid** treating the combined score as the only signal. A model with high combined rho but structural consistency below 0.35 will produce explanations that rephrase erratically, degrading user trust even if meaning is preserved.

## Error Handling

- **LLM returns empty or refusal responses for perturbed inputs**: Some perturbations (especially missing values at severity 5) can produce histories so sparse the LLM refuses to explain. Treat these as robustness failures with score 0.0 for all metrics, and log them separately as "generation failures."
- **Keyword extraction yields empty sets**: If POS tagging extracts no keywords from very short explanations, Jaccard similarity is undefined. Default to 0.0 and flag the pair for manual review.
- **BLEU score is 0.0 for all pairs**: This typically means the model is paraphrasing heavily rather than maintaining structural patterns. This is a legitimate finding, not an error — report it as low structural robustness.
- **High variance across users**: If robustness scores vary wildly (std > 0.2), stratify results by user history length and interaction diversity. Short histories are inherently more sensitive to perturbation.
- **Perturbation produces an identical history**: At severity level 1, some perturbations (especially temporal shuffle on short histories) may not change the history. Filter these trivial pairs from the evaluation.

## Limitations

- The framework evaluates **explanation consistency**, not explanation correctness. A model that always generates the same wrong explanation will score high on robustness. Combine with relevance/faithfulness evaluation.
- Bag-of-words semantic similarity is a lightweight proxy. For production evaluation, consider supplementing with embedding-based similarity, though the paper demonstrates BoW captures meaningful signal.
- The five perturbation types cover the most common real-world noise sources but do not model adversarial attacks (e.g., deliberate injection of misleading history items for manipulation).
- Equal metric weighting assumes all dimensions matter equally. In practice, semantic preservation may matter more than structural consistency for end-user trust — validate weights with user studies.
- The framework assumes English-language explanations. Keyword extraction (POS tagging) and BLEU computation may need adaptation for other languages.
- Evaluation cost scales as O(users x items x perturbation_types x severity_levels) LLM calls. For large-scale deployment, sample strategically rather than exhaustively evaluating.

## Reference

**Paper**: [RobustExplain: Evaluating Robustness of LLM-Based Explanation Agents for Recommendation](https://arxiv.org/abs/2601.19120v3) — Zhang, Zhao, Friedman, Chu (2026). Look for Section 3 (perturbation definitions and metric formulas), Table 2 (per-model robustness baselines), and Table 4 (severity degradation analysis) for the core methodology and reference numbers.