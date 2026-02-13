---
name: "cross-lingual-stability-judges-under"
description: "Detect and fix cross-lingual evaluation instabilities in LLM-as-a-judge pipelines. Use when: 'audit my multilingual eval pipeline', 'check if my LLM judge is stable across languages', 'set up cross-lingual evaluation', 'calibrate judge scoring for non-English languages', 'diagnose ranking inversions in multilingual benchmarks', 'build controlled generation tests for eval reliability'."
---

# Cross-Lingual LLM Judge Stability Auditing

This skill enables Claude to diagnose, measure, and fix evaluation instability when LLM judges score outputs across multiple languages. Based on Chung & Freienthal (2026), it implements a controlled generation protocol that isolates measurement noise from genuine performance differences: by generating semantically identical content across languages with fixed parameters, you can detect which evaluation dimensions (surface metrics vs. pragmatic judgments) break down in cross-lingual transfer. The core insight is that coherence and instruction-following scores from zero-shot LLM judges exhibit rank inversions and near-zero correlations across morphologically rich languages, while lexical diversity and semantic similarity remain stable.

## When to Use

- When the user is building a multilingual evaluation pipeline and wants to know if their LLM judge produces reliable rankings across languages
- When the user observes inconsistent model rankings between languages and needs to determine whether the problem is the model or the evaluator
- When the user asks to set up LLM-as-a-judge scoring for non-English or morphologically rich languages (Finnish, Estonian, Hungarian, Turkish, Korean, Japanese, etc.)
- When the user wants to calibrate automatic evaluation metrics against human annotations for a specific target language
- When the user needs to audit an existing multilingual benchmark for measurement instability before publishing results
- When the user asks to generate controlled synthetic test data to stress-test an evaluation framework

## Key Technique

**Controlled generation as a diagnostic probe.** The central method is to eliminate variation from the thing being evaluated so that any score differences must come from the evaluator. You generate synthetic dialogues using identical prompts, temperature, system instructions, and topic parameters across every target language. When generation conditions are held constant, a reliable judge should produce stable model rankings regardless of language. Ranking inversions under these conditions are direct evidence of evaluator failure, not model failure.

**Two-tier metric taxonomy.** The paper distinguishes surface-level metrics (lexical diversity via type-token ratio, surface similarity via character n-gram overlap, semantic similarity via multilingual embeddings) from pragmatic metrics (coherence, instruction-following, grammaticality, readability, fluency). The surface metrics maintain cross-language ranking stability. The pragmatic metrics -- which require discourse-level understanding -- exhibit rank inversions and near-zero Kendall tau correlations across language pairs. This means you cannot trust zero-shot LLM judges for discourse-level assessment in morphologically rich languages without language-specific calibration.

**Language-specific calibration against human baselines.** The fix is not to abandon LLM judges but to collect a small targeted set of human annotations per language (the paper uses Estonian native speaker annotations as reference) and calibrate judge scores against that ground truth. Measure Kendall tau and Spearman rho between judge rankings and human rankings per evaluation dimension. Dimensions where correlation drops below an acceptable threshold (e.g., tau < 0.3) need either prompt adaptation, few-shot examples in the target language, or replacement with a language-specific metric.

## Step-by-Step Workflow

1. **Define the evaluation dimensions.** Separate your metrics into surface-level (lexical diversity, n-gram overlap, embedding similarity) and pragmatic (coherence, instruction-following, fluency, grammaticality). Label each dimension explicitly -- surface metrics will likely transfer; pragmatic metrics need validation.

2. **Build the controlled generation protocol.** Create a generation config that fixes all parameters: system prompt, topic/domain, temperature, max tokens, few-shot examples, and any structural constraints. The config must be identical across all target languages -- only the language instruction changes. Use a structured schema:
   ```python
   generation_config = {
       "domain": "customer_support",
       "industry": "telecommunications",
       "problem_type": "billing_dispute",
       "channel": "chat",
       "temperature": 0.7,
       "max_tokens": 512,
       "models": ["gpt-4.1-mini", "claude-sonnet-4-20250514"],
       "languages": ["et", "fi", "hu"],  # target languages
       "num_samples": 50  # per model per language
   }
   ```

3. **Generate synthetic parallel data.** For each (model, language) pair, generate N dialogues using the fixed config. Store outputs in JSONL with metadata tracking the generation parameters. Use liteLLM or a similar abstraction layer to switch providers without changing the evaluation interface:
   ```bash
   python -m src.conversation_generator \
     --provider openai -m gpt-4.1-mini \
     --lang et fi hu --samples 50
   ```

4. **Run surface-level metrics.** Compute lexical diversity (type-token ratio, hapax legomena ratio), surface similarity (character/word n-gram overlap between parallel outputs), and semantic similarity (cosine distance using multilingual embeddings like LaBSE or multilingual-e5). Record per-model scores grouped by language.

5. **Run LLM-as-a-judge scoring.** Score each generated dialogue on pragmatic dimensions using a structured rubric. Use explicit 0-N scales with anchor descriptions per level. Score grammaticality (0-4), readability (0-4), content coherence (0-3), and fluency (0-3). Capture both numeric scores and free-text explanations:
   ```bash
   python -m src.llm_judge \
     --provider openai -m gpt-4.1 \
     --input data/conversations.json \
     --output results/judge_scores.jsonl
   ```

6. **Compute ranking stability.** For each evaluation dimension, rank the generator models within each language, then compute pairwise Kendall tau and Spearman rho across all language pairs. Use bootstrap confidence intervals (2000+ iterations) and permutation tests (5000+ iterations) for significance:
   ```python
   from scipy.stats import kendalltau, spearmanr
   # For each dimension, compare model rankings between language pairs
   tau, p_value = kendalltau(ranks_estonian, ranks_finnish)
   ```

7. **Identify unstable dimensions.** Flag any dimension where tau < 0.3 or where rank inversions occur (model A > model B in language X, but model B > model A in language Y). These dimensions are unreliable for cross-lingual comparison without calibration.

8. **Collect targeted human annotations.** For each unstable dimension, collect 30-50 human judgments per target language from native speakers. This is the minimum viable calibration set. Compute inter-annotator agreement (Krippendorff's alpha) to verify annotation quality.

9. **Calibrate or replace unstable metrics.** For each flagged dimension, compute correlation between judge scores and human scores. If correlation is acceptable after seeing the human distribution (tau > 0.5), apply a linear calibration offset per language. If correlation is low, either (a) add language-specific few-shot examples to the judge prompt, (b) switch to a judge model with stronger multilingual training, or (c) replace the LLM judge dimension with a specialized metric for that language.

10. **Document and version the stability report.** Output a per-dimension, per-language-pair stability matrix showing tau values, confidence intervals, and calibration status. This report accompanies the evaluation results and flags which scores are trustworthy.

## Concrete Examples

**Example 1: Auditing a multilingual chatbot evaluation pipeline**

User: "I'm evaluating three chatbot models across 5 languages (English, Finnish, Estonian, Turkish, Korean) using GPT-4 as a judge. The rankings keep changing between languages. Is my judge broken?"

Approach:
1. Set up controlled generation: generate 50 customer-support dialogues per model per language using identical parameters (same domain, temperature, system prompt).
2. Compute surface metrics (type-token ratio, multilingual embedding similarity) and pragmatic metrics (coherence 0-3, instruction-following 0-3) using the same GPT-4 judge.
3. Build a 5x5 language-pair Kendall tau matrix for each metric dimension.
4. Identify that coherence and instruction-following show tau < 0.2 between Finnish-Turkish and Estonian-Korean, while lexical diversity shows tau > 0.8 across all pairs.
5. Recommend collecting 40 human annotations each for Finnish, Turkish, and Korean on coherence/instruction-following to calibrate.

Output:
```
Stability Report
=================
Dimension              | en-fi | en-et | en-hu | fi-et | fi-hu | et-hu
-----------------------|-------|-------|-------|-------|-------|------
Lexical diversity      | 0.87  | 0.82  | 0.85  | 0.91  | 0.83  | 0.88
Semantic similarity    | 0.79  | 0.75  | 0.81  | 0.84  | 0.77  | 0.80
Coherence              | 0.34  | 0.28  | 0.41  | 0.62  | 0.19  | 0.22  [!]
Instruction-following  | 0.21  | 0.15  | 0.29  | 0.45  | 0.11  | 0.18  [!]

[!] = Unstable: tau < 0.3, calibration required before cross-lingual comparison
```

**Example 2: Building a controlled generation test suite from scratch**

User: "I want to test whether my custom LLM judge is reliable for scoring Estonian and Hungarian outputs. How do I set this up?"

Approach:
1. Define a generation config fixing domain (e.g., tech support), problem type, channel, temperature=0.7, and 3 generator models to rank.
2. Generate 50 parallel dialogues per model in Estonian, Hungarian, and English (English as a high-resource control).
3. Score all outputs with the custom judge on grammaticality (0-4), readability (0-4), coherence (0-3), fluency (0-3).
4. Compute pairwise Kendall tau for model rankings between en-et, en-hu, and et-hu.
5. If en-et coherence tau is 0.85 but et-hu coherence tau is 0.15, the judge transfers from English to Estonian but not from Estonian to Hungarian for that dimension.
6. Collect 30 human annotations for Hungarian coherence and recalibrate.

Output:
```python
# controlled_gen_config.py
CONFIG = {
    "domain": "tech_support",
    "problem_types": ["connectivity", "billing", "account_access"],
    "channel": "live_chat",
    "temperature": 0.7,
    "max_tokens": 512,
    "generator_models": [
        "gpt-4.1-mini",
        "claude-sonnet-4-20250514",
        "mistral-large-latest"
    ],
    "languages": ["en", "et", "hu"],
    "samples_per_cell": 50,
    "judge_model": "custom-judge-v2",
    "judge_dimensions": {
        "grammaticality": {"scale": [0, 4], "type": "surface-adjacent"},
        "readability": {"scale": [0, 4], "type": "surface-adjacent"},
        "coherence": {"scale": [0, 3], "type": "pragmatic"},
        "fluency": {"scale": [0, 3], "type": "pragmatic"},
    }
}
```

**Example 3: Diagnosing a specific ranking inversion**

User: "Model A scores higher than Model B in Finnish coherence, but Model B scores higher in Estonian coherence. Same judge, same prompt. What's going on?"

Approach:
1. Confirm this is a ranking inversion, not noise: check whether the score difference is statistically significant via bootstrap CI on the mean score gap.
2. Verify generation was controlled: same system prompt, temperature, domain, and sample size for both languages.
3. Examine the judge's free-text explanations for Finnish vs. Estonian coherence scores. Look for signs the judge is conflating grammatical complexity with incoherence (common in agglutinative languages).
4. Compute the judge's self-consistency: re-score a 20% subsample and measure score-rescore correlation per language. If Finnish self-consistency is lower, the judge is uncertain in Finnish.
5. Collect 20 Estonian and 20 Finnish native speaker coherence annotations. Compute judge-human tau per language. If Finnish judge-human tau is 0.15 but Estonian is 0.55, the judge fails specifically on Finnish coherence.
6. Recommend: add 5 Finnish few-shot examples with coherence annotations to the judge prompt, or switch to a Finnish-tuned judge for that dimension.

Output:
```
Inversion Diagnosis: Model A vs Model B, Coherence
===================================================
Language  | Model A mean | Model B mean | Gap CI (95%)     | Judge-Human tau
----------|-------------|-------------|------------------|----------------
Finnish   | 2.4         | 2.1         | [0.1, 0.5]       | 0.15 [UNRELIABLE]
Estonian  | 1.9         | 2.3         | [-0.6, -0.2]     | 0.55 [OK]

Diagnosis: Judge scoring is unreliable for Finnish coherence (tau=0.15).
The ranking inversion reflects judge instability, not a real performance difference.
Action: Calibrate Finnish coherence with native speaker annotations + few-shot examples.
```

## Best Practices

- **Do:** Always include a high-resource language (English) as a control when testing cross-lingual stability. If rankings are unstable even between English and your target, the judge prompt itself may be the issue.
- **Do:** Separate surface-level metrics from pragmatic metrics in your reporting. Surface metrics (lexical diversity, embedding similarity) are generally safe to compare cross-lingually; pragmatic metrics (coherence, instruction-following) are not without calibration.
- **Do:** Use bootstrap confidence intervals (2000+ iterations) and permutation tests when computing ranking correlations. Small sample sizes can produce misleading tau values.
- **Do:** Log the judge's free-text explanations alongside numeric scores. Explanations reveal whether the judge is applying language-specific biases (e.g., penalizing agglutinative morphology as "complexity").
- **Avoid:** Assuming zero-shot LLM judge transfer works for discourse-level metrics. The paper shows near-zero correlations for pragmatic judgments across morphologically rich languages.
- **Avoid:** Using a single aggregate score across dimensions. A judge can be stable on grammaticality but unstable on coherence for the same language pair -- aggregation hides this.
- **Avoid:** Treating ranking stability between related languages (e.g., Estonian-Finnish) as evidence of stability for unrelated languages (e.g., Estonian-Korean). Language family proximity does not guarantee transfer.

## Error Handling

- **Low sample size warning:** If fewer than 30 samples per (model, language) cell, tau estimates are unreliable. Warn the user and recommend increasing to 50+.
- **Judge refusal or empty scores:** Some LLM judges refuse to score content in low-resource languages or return uniform scores. Detect this by checking score variance per language -- if stddev < 0.1, the judge is likely not discriminating. Switch to a more multilingual judge model.
- **Human annotation disagreement:** If inter-annotator Krippendorff's alpha < 0.4 on the calibration set, the rubric is ambiguous for that language. Refine anchor descriptions with native-speaker input before calibrating.
- **Metric computation failures:** Multilingual embedding models may have degraded coverage for very low-resource languages. Verify embedding quality by checking that same-language paraphrases have cosine > 0.8 before trusting semantic similarity scores.
- **Controlled generation leakage:** If the generator model refuses to produce content in a target language and falls back to English, the controlled generation assumption is violated. Detect by running a language-ID classifier on all outputs and filtering contaminated samples.

## Limitations

- The controlled generation approach requires the generator models to actually support the target languages. For extremely low-resource languages where generators produce poor output, the "controlled" assumption weakens.
- Calibration requires native speaker annotations, which are expensive and slow to obtain for many languages. The minimum viable set is ~30-50 annotations per dimension per language.
- The framework is validated on Finno-Ugric languages (agglutinative morphology). Transfer to other language families (tonal languages, logographic scripts) needs separate validation.
- Surface-level metric stability does not guarantee that those metrics are *useful* -- they may be stable but uninformative for the evaluation task at hand.
- The method diagnoses instability but does not automatically fix it. Calibration, few-shot prompting, or judge replacement still require human judgment and language expertise.

## Reference

Chung, I., & Freienthal, L. (2026). Cross-Lingual Stability of LLM Judges Under Controlled Generation: Evidence from Finno-Ugric Languages. *First Workshop on Multilingual Multicultural Evaluation (MME), co-located with EACL 2026.* [arXiv:2602.02287](https://arxiv.org/abs/2602.02287v1) | [Code](https://github.com/isaac-chung/cross-lingual-stability-judges)

Key takeaway from the paper: surface metrics transfer cross-lingually but pragmatic judgments (coherence, instruction-following) do not -- use the controlled generation protocol to detect which dimensions of your evaluation pipeline are unreliable before trusting cross-lingual rankings.