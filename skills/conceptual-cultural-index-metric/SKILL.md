---
name: conceptual-cultural-index-metric
description: >
  Compute the Conceptual Cultural Index (CCI) to measure cultural specificity of sentences
  using LLM-based generality estimates across culture sets. Use this skill when users ask to
  "measure cultural specificity", "score how culture-specific a sentence is", "compare cultural
  relevance across countries", "detect culturally loaded content", "evaluate cultural bias in text",
  or "quantify how Japanese/American/etc. a sentence is".
---

# Conceptual Cultural Index (CCI) Metric

This skill enables Claude to compute the Conceptual Cultural Index -- a sentence-level metric that
quantifies how culturally specific a piece of text is to a target culture. CCI works by asking an
LLM to estimate how "common" or "general" a sentence is within each culture in a comparison set,
then computing the difference between the target culture's generality score and the average of all
other cultures. The result is a score in [-1, 1] where positive values indicate target-culture
specificity and values near zero indicate cross-cultural generality.

## When to Use

- When a user wants to measure how culturally specific a sentence, question, or statement is to a particular culture or country
- When building a multilingual/multicultural NLP pipeline that needs to flag or stratify content by cultural specificity
- When evaluating whether benchmark questions (e.g., commonsense QA) are culturally biased toward one culture
- When analyzing a corpus to find the most and least culturally loaded sentences
- When designing localization pipelines and need to identify content requiring cultural adaptation
- When comparing how a concept is perceived across different cultural contexts (e.g., G20 nations)
- When filtering training data to balance cultural representation

## Key Technique

**The core insight:** Rather than asking an LLM to directly rate "how culturally specific is this?" (which produces noisy, poorly calibrated scores), CCI decomposes the problem into per-culture *generality estimates* and derives specificity from the *relative difference*. This indirect approach stabilizes LLM inference and yields 10+ point AUC improvements over direct scoring for culture-specialized models.

**The formula:**

```
CCI(x; t, C) = p_bar_t(x) - (1 / (|C| - 1)) * SUM_{c in C \ {t}} p_bar_c(x)
```

Where `x` is the input sentence, `t` is the target culture, `C` is the full comparison culture set, and `p_bar_c(x)` is the averaged generality score for culture `c`. Each generality score is obtained by prompting an LLM to rate how "common" the sentence is within each culture on a [0, 1] scale, then averaging across N=3 independent runs to reduce variance. All cultures are queried in a single prompt, with the LLM returning a JSON object mapping culture names to scores.

**Why relative generality works:** A sentence like "We eat osechi on New Year's Day" gets a high generality score for Japan but low scores for most other countries. The difference produces a strong positive CCI. A sentence like "Water boils at 100 degrees Celsius" gets uniformly high scores everywhere, producing CCI near zero. By controlling which cultures appear in the comparison set `C`, users can sharpen or broaden the cultural contrast -- for example, including neighboring East Asian cultures reduces CCI for pan-Asian concepts, while a global G20 set maximizes the contrast.

## Step-by-Step Workflow

1. **Define the target culture and comparison set.** Choose the target culture `t` (e.g., "Japan") and a comparison set `C` (e.g., G20 nations: USA, UK, France, Germany, Japan, China, South Korea, India, Brazil, etc.). The comparison set controls the cultural "lens" -- broader sets detect globally unique content; narrower sets detect regionally unique content.

2. **Prepare the input sentences.** Collect the sentences to score. Each sentence is evaluated independently. Ensure sentences are in a language the LLM can process well (the original paper uses Japanese sentences with multilingual models).

3. **Construct the generality estimation prompt.** Build a prompt that presents the sentence and asks the LLM to rate how commonly known, understood, or relevant it is within *each* culture in `C`. Request output as a JSON object mapping culture names to float scores in [0, 1], where 1.0 means "universally known in this culture" and 0.0 means "completely unknown."

   ```
   Example prompt template:
   "For the following sentence, rate how commonly known or relevant it is
   within each of the listed cultures. Return a JSON object mapping each
   culture to a score between 0.0 (completely unknown/irrelevant) and
   1.0 (universally known/common).

   Cultures: {culture_list}
   Sentence: {sentence}

   Respond ONLY with valid JSON. Example format:
   {"Japan": 0.95, "USA": 0.2, "France": 0.15, ...}"
   ```

4. **Run N=3 independent LLM calls per sentence.** For each sentence, send the prompt 3 times (with temperature > 0 to get variation) and collect the per-culture score vectors. This averaging reduces LLM scoring noise.

5. **Compute averaged generality scores.** For each culture `c` in `C`, average the N scores: `p_bar_c(x) = (1/N) * sum(scores_for_c_across_runs)`.

6. **Calculate CCI.** Apply the formula: subtract the mean of all non-target culture scores from the target culture's score.

   ```python
   target_score = averaged_scores[target_culture]
   other_scores = [averaged_scores[c] for c in cultures if c != target_culture]
   cci = target_score - (sum(other_scores) / len(other_scores))
   ```

7. **Interpret the result.** CCI in [-1, 1]:
   - **CCI > 0.3**: Strongly specific to the target culture
   - **CCI ~ 0.0**: Culturally general / universal
   - **CCI < -0.3**: More specific to *other* cultures in the comparison set

8. **Optionally adjust the comparison set.** If results seem off, try adding or removing culturally "neighboring" countries. Including neighbors (e.g., adding South Korea and China when targeting Japan) reduces CCI for pan-regional concepts, giving a stricter measure of *uniquely* target-culture content.

9. **Use CCI scores downstream.** Apply the scores to stratify benchmarks by cultural difficulty, filter datasets, flag content for localization review, or rank sentences by cultural specificity.

## Concrete Examples

**Example 1: Scoring Japanese cultural specificity of sentences**

User: "I have these three sentences and want to know how culturally Japanese they are:
1. 'Osechi cuisine is enjoyed during the New Year celebrations.'
2. 'Water boils at 100 degrees Celsius at sea level.'
3. 'Hanami parties are held under cherry blossom trees in spring.'"

Approach:
1. Set target culture = "Japan", comparison set = G20 nations
2. For each sentence, prompt the LLM 3 times with the generality estimation template
3. Parse JSON responses and average across runs
4. Compute CCI for each sentence

Output:
```
Sentence 1 (Osechi):
  Japan: 0.95  |  USA: 0.12  |  France: 0.08  |  Brazil: 0.05  |  avg_others: 0.10
  CCI = 0.95 - 0.10 = +0.85  --> Highly Japan-specific

Sentence 2 (Boiling water):
  Japan: 0.98  |  USA: 0.97  |  France: 0.98  |  Brazil: 0.96  |  avg_others: 0.97
  CCI = 0.98 - 0.97 = +0.01  --> Culturally universal

Sentence 3 (Hanami):
  Japan: 0.93  |  USA: 0.25  |  France: 0.18  |  South Korea: 0.55  |  avg_others: 0.22
  CCI = 0.93 - 0.22 = +0.71  --> Strongly Japan-specific
```

**Example 2: Evaluating cultural bias in a QA benchmark**

User: "I have a commonsense QA dataset with 500 questions. I want to find which questions are culturally biased toward American culture."

Approach:
1. Set target culture = "USA", comparison set = G20 nations
2. Batch-process all 500 questions through the CCI pipeline (3 LLM calls each)
3. Rank questions by CCI score descending
4. Report distribution statistics and the most/least culturally biased items

Output:
```
CCI Distribution (target: USA, comparison: G20):
  Mean CCI:   +0.12
  Median CCI: +0.08
  Std Dev:     0.31

Top 5 most US-specific questions (highest CCI):
  1. "What do Americans typically eat on Thanksgiving?" CCI = +0.88
  2. "Which sport is played during the Super Bowl?"     CCI = +0.82
  3. "What is a 401(k) retirement plan?"                CCI = +0.79
  ...

Bottom 5 (most universal):
  1. "What color is the sky on a clear day?"            CCI = +0.00
  2. "How many days are in a week?"                     CCI = +0.01
  ...

Recommendation: 47 questions (9.4%) have CCI > 0.5, indicating strong US cultural
bias. Consider replacing or annotating these for non-US evaluation contexts.
```

**Example 3: Comparing cultural scope effects**

User: "How does including neighboring cultures change the CCI score for 'Kimchi is eaten with most meals'?"

Approach:
1. Score with global set (G20): target = "South Korea"
2. Score with East Asia focus: C = {South Korea, Japan, China, USA, France}
3. Compare how the narrower set changes the CCI

Output:
```
Global set (G20, 19 comparisons):
  South Korea: 0.97  |  avg_others: 0.18  |  CCI = +0.79

East Asia focus (4 comparisons):
  South Korea: 0.97  |  Japan: 0.62  |  China: 0.55  |  USA: 0.30  |  France: 0.15
  avg_others: 0.41  |  CCI = +0.56

Interpretation: Including East Asian neighbors reduces CCI from 0.79 to 0.56 because
kimchi has significant recognition in Japan and China. The global set treats it as
highly Korea-specific; the regional set reveals it's more of an East Asian concept
with strongest ties to Korea.
```

## Implementation (Python)

```python
import json
import statistics

def compute_cci(
    sentence: str,
    target_culture: str,
    cultures: list[str],
    llm_call_fn,  # function(prompt: str) -> str
    n_runs: int = 3,
) -> dict:
    """Compute Conceptual Cultural Index for a sentence."""
    prompt = (
        "For the following sentence, rate how commonly known or relevant "
        "it is within each listed culture. Return ONLY a JSON object mapping "
        "each culture to a float between 0.0 (completely unknown) and "
        "1.0 (universally known).\n\n"
        f"Cultures: {', '.join(cultures)}\n"
        f"Sentence: {sentence}\n\n"
        "Respond with valid JSON only."
    )

    all_scores = {c: [] for c in cultures}
    for _ in range(n_runs):
        response = llm_call_fn(prompt)
        scores = json.loads(response)
        for c in cultures:
            all_scores[c].append(float(scores[c]))

    averaged = {c: statistics.mean(all_scores[c]) for c in cultures}
    target_score = averaged[target_culture]
    other_scores = [averaged[c] for c in cultures if c != target_culture]
    cci = target_score - statistics.mean(other_scores)

    return {
        "cci": round(cci, 4),
        "target_score": round(target_score, 4),
        "avg_other_score": round(statistics.mean(other_scores), 4),
        "per_culture_scores": {c: round(v, 4) for c, v in averaged.items()},
    }
```

## Best Practices

- **Do** use at least N=3 runs and average scores. Single LLM calls have high variance for numeric ratings; averaging is essential for stable CCI values.
- **Do** query all cultures in a single prompt rather than one prompt per culture. This gives the LLM joint context and produces more consistent relative scores.
- **Do** choose a comparison set that matches your analytical goal. Use G20 for global specificity; use regional neighbors for regional specificity.
- **Do** validate on known sentences first. Score a few obviously culture-specific and obviously universal sentences to calibrate your setup before processing a full corpus.
- **Avoid** using CCI with models that have weak multilingual capabilities. The generality estimation requires cultural knowledge the model may lack. Prefer models with strong coverage of the target culture's language and knowledge.
- **Avoid** interpreting CCI as a binary label. It is a continuous metric -- use it for ranking, stratification, and threshold-based filtering rather than hard classification.
- **Avoid** comparing CCI scores computed with different comparison sets. The score is relative to the comparison cultures, so changing `C` changes the scale.

## Error Handling

- **LLM returns invalid JSON:** Wrap the JSON parsing in a try/except. Retry with a stricter prompt that includes an explicit JSON example. After 3 failures, skip the sentence and log a warning.
- **LLM omits a culture from the response:** Check that all cultures in `C` appear in the parsed JSON. If missing, either retry or assign 0.5 (neutral) as a fallback, but flag the result as degraded.
- **Scores outside [0, 1]:** Clamp values to [0, 1] before computing CCI. Some LLMs occasionally return values like 1.1 or -0.05.
- **All cultures receive identical scores:** This produces CCI = 0, which is correct for genuinely universal content but may also indicate the LLM failed to differentiate. Inspect the raw scores; if all are exactly 0.5, the model likely defaulted rather than reasoning.
- **Target culture not in comparison set:** Raise an error -- `t` must be a member of `C` for the formula to be valid.

## Limitations

- **LLM knowledge dependency:** CCI quality depends entirely on the LLM's cultural knowledge. Models trained primarily on English text may lack nuanced understanding of non-Western cultures, producing less reliable generality estimates for those cultures.
- **Language effects:** The paper validates on Japanese sentences. Scoring sentences in a language the LLM handles poorly will degrade results. For best results, use a model strong in the sentence's language.
- **Sentence-level only:** CCI scores individual sentences, not paragraphs or documents. For longer texts, score sentences individually and aggregate (e.g., mean or max CCI).
- **Not a ground truth:** CCI is an estimate based on LLM perception of cultural commonality. It does not measure actual cultural knowledge prevalence in populations.
- **Comparison set sensitivity:** Results change with the comparison set. There is no single "correct" set -- the choice is an analytical decision that should be documented.
- **Cost:** Each sentence requires N * 1 LLM calls (N=3 recommended). For large corpora, this adds up. Consider batching sentences if the LLM API supports it, or reducing N to 1 for exploratory analysis.

## Reference

Ohashi, T. & Iyatomi, H. (2026). *Conceptual Cultural Index: A Metric for Cultural Specificity via Relative Generality.* First Workshop on Multilingual Multicultural Evaluation (MME) @ EACL 2026. [arXiv:2602.09444](https://arxiv.org/abs/2602.09444) -- See Section 3 for the formal CCI definition and Section 4 for validation on the 400-sentence evaluation set with AUC comparisons.