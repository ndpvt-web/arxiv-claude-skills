---
name: "urdubench-urdu-reasoning-benchmark"
description: "Build high-quality reasoning benchmarks for Urdu and other low-resource languages using contextually ensembled translation with human-in-the-loop validation. Triggers: 'translate benchmark to Urdu', 'evaluate LLM in Urdu', 'low-resource language benchmark', 'Urdu reasoning evaluation', 'ensembled translation pipeline', 'multilingual benchmark creation'"
---

# Contextually Ensembled Translation for Low-Resource Language Reasoning Benchmarks

This skill enables Claude to apply the UrduBench methodology: a multi-system translation ensemble with automated fusion, heuristic validation, and human-in-the-loop refinement to create reasoning benchmarks in Urdu and other low-resource languages. The technique addresses the core problem that naive machine translation destroys the structural and semantic integrity of reasoning tasks — especially multi-step math, symbolic logic, and commonsense QA — by combining four independent translation systems, fusing their outputs with an LLM judge, and enforcing strict post-processing heuristics to catch translation failures before they contaminate benchmarks.

## When to Use

- When the user wants to translate an English reasoning benchmark (math, QA, commonsense) into Urdu or another low-resource language
- When evaluating LLM reasoning performance in Urdu across different prompting strategies (direct, CoT, few-shot+CoT)
- When building a translation pipeline that preserves LaTeX math notation, multiple-choice structure, and semantic alignment
- When the user needs to assess language consistency — whether a model actually responds in the target language during reasoning
- When designing a quality-controlled translation workflow that combines automated ensemble methods with human annotator review
- When comparing model architectures and scales for multilingual reasoning tasks

## Key Technique

**Contextually Ensembled Translation** solves the fragility of single-system translation by running four independent translation models (e.g., IndicTrans2, NLLB-200, Qwen-3-30B, Gemini-2.5-Pro) on every source instance, then using an LLM judge (e.g., GPT-5.1) to either synthesize the best elements into a superior translation or select the highest-quality candidate outright. The critical insight is that different translation systems excel at different aspects — one may handle formal register well, another may preserve idiomatic phrasing — so ensembling captures complementary strengths that no single system provides.

**Structure preservation** is equally important. For multiple-choice benchmarks (CommonSenseQA, OpenBookQA), questions and all answer choices are concatenated into a single translation unit rather than translated independently. This prevents semantic drift between a question stem and its options. After translation, components are programmatically separated back into their original structure. For math benchmarks (MATH-500, MGSM), all LaTeX expressions inside `$` or `$$` delimiters are explicitly excluded from translation, preserving mathematical notation exactly.

**Automated heuristic validation** (the post-processing gate) catches systematic failures before human review: empty translations, missing answer options, abnormally short outputs (<10 characters), invalid answer keys, and excessive residual English (>30% English character ratio). Flagged instances are reprocessed. This reduces the human annotation burden to genuine quality judgments rather than mechanical error detection.

## Step-by-Step Workflow

1. **Select source benchmarks and categorize by task type.** Identify whether each benchmark requires multi-step reasoning (MGSM, MATH-500), commonsense reasoning (CommonSenseQA), or knowledge retrieval (OpenBookQA). Task type determines translation constraints and prompting strategy.

2. **Prepare translation-safe source instances.** For multiple-choice tasks, concatenate each question stem with all answer choices into a single string using a consistent delimiter (e.g., `\n(A) ... \n(B) ...`). For math tasks, identify and tag all LaTeX expressions (`$...$`, `$$...$$`) with preservation markers so translators leave them untouched.

3. **Run parallel translation through 4 independent systems.** Send each prepared instance to all four translation models simultaneously. Use translation prompts that specify: (a) translate only natural language text, (b) preserve mathematical notation exactly, (c) maintain formal academic register, (d) preserve exact structure, line breaks, and spacing.

4. **Fuse translations with an LLM judge.** Present all four candidate translations alongside the original English source to a strong LLM. Instruct it to either synthesize an improved Urdu translation combining the best elements, or select the single best candidate. The judge prompt should emphasize accuracy, naturalness, semantic preservation, and structural fidelity.

5. **Apply heuristic post-processing validation.** Programmatically check each fused translation for: (a) empty or null output, (b) missing Urdu question text or answer options, (c) question length below 10 characters, (d) invalid or missing answer keys, (e) English character ratio exceeding 30%. Flag failures for reprocessing through the pipeline.

6. **Programmatically separate translated components.** For concatenated multiple-choice instances, split the fused translation back into question stem and individual answer options using the original delimiters. Validate that the correct number of options was preserved.

7. **Conduct human-in-the-loop validation.** Present native Urdu speakers (fluent in English) with the ensemble candidates and the fused output. Annotators select or edit the best translation using a rubric covering: translation accuracy, naturalness, semantic preservation, cultural appropriateness, and technical term handling.

8. **Design prompting strategies per task type.** For math reasoning: test Direct (implicit reasoning), Chain-of-Thought (explicit intermediate steps), and Few-Shot+CoT (3 Urdu exemplars before the target). For commonsense QA: use Direct prompting. For advanced math: use CoT zero-shot exclusively.

9. **Evaluate with language consistency measurement.** After generating model responses, strip special tokens and math notation, then run language identification on cleaned text. Count a response as language-consistent only if Urdu is the sole detected language. Compute: `Language Consistency (%) = (# Urdu-only answers) / (# total answers) * 100`. Correlate consistency with accuracy.

10. **Analyze results across five dimensions.** Report performance stratified by: (a) dataset, (b) difficulty level (L1-L5 for math), (c) model architecture, (d) model scale, and (e) language consistency. Identify where multi-step and symbolic reasoning degrade most under linguistic transfer.

## Concrete Examples

**Example 1: Translating a math benchmark to Urdu**

User: "I have a set of 500 math problems in English with LaTeX notation. Help me create a high-quality Urdu version."

Approach:
1. Parse each problem, tagging LaTeX blocks (`$x^2 + 3x = 0$`) as untranslatable.
2. Send each problem text (with LaTeX masked) to four translation APIs in parallel.
3. Collect all four Urdu candidates per problem. Feed them plus the English original to an LLM judge with the prompt:

```
You are given an English math problem and four Urdu translations.
Compare all candidates against the original for:
- Accuracy of meaning
- Natural Urdu phrasing (formal academic register)
- Preservation of ALL LaTeX expressions exactly as-is
- Structural fidelity (line breaks, spacing)

Either synthesize the best Urdu translation or select the best candidate.
Output ONLY the final Urdu translation.

English: {source}
Translation A (IndicTrans2): {t1}
Translation B (NLLB-200): {t2}
Translation C (Qwen): {t3}
Translation D (Gemini): {t4}
```

4. Run heuristic checks: verify LaTeX expressions survived intact, Urdu text is present, English ratio < 30%.
5. Output the validated Urdu benchmark in the original format with LaTeX preserved.

Output:
```json
{
  "problem_id": 42,
  "english": "If $x^2 + 3x - 10 = 0$, find all values of $x$.",
  "urdu": "اگر $x^2 + 3x - 10 = 0$ ہو تو $x$ کی تمام قدریں معلوم کریں۔",
  "source_translations": ["indictr2", "nllb200", "qwen", "gemini"],
  "selected_by": "fusion",
  "english_ratio": 0.08,
  "validation": "passed"
}
```

**Example 2: Evaluating an LLM on Urdu commonsense reasoning**

User: "I want to test how well Gemma-3-12B handles commonsense QA in Urdu."

Approach:
1. Load the translated CommonSenseQA benchmark (question + 5 answer choices in Urdu).
2. Format each instance as a direct prompt — no chain-of-thought for commonsense tasks:

```
سوال: {urdu_question}
(الف) {option_a}
(ب) {option_b}
(ج) {option_c}
(د) {option_d}
(ہ) {option_e}

صرف صحیح جواب کا حرف لکھیں۔
```

3. Run inference, extract the predicted answer letter from each response.
4. Measure accuracy against gold labels and compute language consistency.
5. Report results broken down by question category.

Output:
```
Model: Gemma-3-12B-it
Dataset: CommonSenseQA (Urdu)
Accuracy: 72.4%
Language Consistency: 96.8%
Note: High Urdu adherence correlates with strong commonsense performance.
```

**Example 3: Building a translation pipeline for a new low-resource language**

User: "I want to adapt this approach to create a Bengali reasoning benchmark."

Approach:
1. Identify 4 translation systems with Bengali support (e.g., IndicTrans2, NLLB-200, Google Translate API, a Bengali-capable LLM).
2. Adapt the heuristic validation thresholds: set minimum question length appropriate for Bengali script, configure the language detection model for Bengali, set English residual ratio threshold at 30%.
3. Recruit native Bengali speakers and adapt the annotation rubric — replace Urdu-specific cultural notes with Bengali equivalents.
4. Follow the same 10-step pipeline: concatenate MCQ components, parallel-translate, LLM-fuse, heuristic-filter, human-validate.
5. For math tasks, preserve LaTeX identically — this step is language-agnostic.
6. Test prompting strategies: start with Direct and CoT, add few-shot only if you have high-quality Bengali exemplars.

Output: A Bengali reasoning benchmark suite covering MGSM, MATH-500, CommonSenseQA, and OpenBookQA with validated translations and baseline model evaluations.

## Best Practices

- **Do:** Concatenate question stems with answer options before translation to preserve semantic alignment. Translating them independently causes drift between the question and its choices.
- **Do:** Preserve all mathematical notation (LaTeX, variables, formulas) verbatim. Explicitly instruct every translation system: "NEVER translate or modify ANY LaTeX math expressions."
- **Do:** Measure language consistency as a first-class metric alongside accuracy. Models that code-switch mid-reasoning (dropping into English during Urdu CoT) show degraded reasoning performance.
- **Do:** Use Chain-of-Thought prompting for multi-step math — it consistently outperforms direct prompting across nearly all model architectures for reasoning tasks.
- **Avoid:** Relying on a single translation system. Single-system translations systematically miss errors that ensembling catches. Even the best individual system (Gemini-2.5-Pro) benefits from ensemble cross-checking.
- **Avoid:** Assuming larger models automatically perform better in low-resource languages. Multilingual training data exposure and language-specific fine-tuning matter more than raw parameter count (Gemma-3-4B outperformed several larger models on Urdu).
- **Avoid:** Using few-shot+CoT indiscriminately. It helps strong multilingual models but causes regressions in smaller or less multilingual models that cannot parse Urdu exemplars reliably.

## Error Handling

| Failure Mode | Detection | Recovery |
|---|---|---|
| Empty or null translation | Heuristic check: output is empty/whitespace | Re-run through pipeline with fallback to best single-system output |
| LaTeX corruption | Regex comparison of `$...$` blocks pre/post translation | Restore original LaTeX from source, re-translate surrounding text only |
| Missing answer options | Count delimiters in translated MCQ output | Re-concatenate and re-translate the full question+options unit |
| Excessive English residual | English character ratio > 30% | Flag for manual review; likely indicates the translation system failed on domain-specific terms |
| Language inconsistency in model output | Language ID detects non-Urdu text in reasoning chain | Report as a metric; consider adjusting system prompt to reinforce target language |
| Semantic drift between question and options | Human annotator flags during validation | Re-translate as a single concatenated unit; do not translate options independently |

## Limitations

- **Human annotation bottleneck:** The pipeline requires native speakers of the target language for final validation. For truly low-resource languages, finding qualified annotators is the primary constraint — the automated pipeline reduces but does not eliminate this need.
- **Translation system availability:** The ensemble approach requires at least 3-4 translation systems with reasonable quality for the target language. Languages with only one viable translator cannot benefit from ensembling.
- **Reasoning-specific:** This methodology targets reasoning benchmarks (math, commonsense QA, logic). It is not designed for generation-quality benchmarks, fluency evaluation, or tasks where creative language use matters more than structural fidelity.
- **Cultural adaptation gap:** Translation preserves the original benchmark's cultural context (Western-centric questions in CommonSenseQA). Some commonsense questions may not transfer culturally even with perfect linguistic translation.
- **Cost:** Running four translation systems plus an LLM judge per instance is expensive at scale. For a 10K-instance benchmark, budget for ~50K translation API calls plus ~10K judge calls.

## Reference

**Paper:** [UrduBench: An Urdu Reasoning Benchmark using Contextually Ensembled Translations with Human-in-the-Loop](https://arxiv.org/abs/2601.21000v1) — Look for: Algorithm 1 (heuristic post-processing), Table 5 (translation system selection rates), Table 3 (model performance comparison), and the translation prompt templates in the appendices.