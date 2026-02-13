---
name: "bias-ear-listener-assessing"
description: "Assess and audit bias in audio/speech language models using the BiasInEar framework. Evaluate multimodal LLM robustness across linguistic, demographic, and positional perturbations with four complementary metrics (accuracy, entropy, APES, Fleiss' kappa). Use when: 'audit my audio model for bias', 'test speech LLM fairness', 'measure option order sensitivity', 'evaluate accent robustness', 'check if my voice AI is biased', 'benchmark multilingual speech model consistency'."
---

This skill enables Claude to design, implement, and execute systematic bias audits for audio language models (speech-integrated LLMs/MLLMs) using the BiasInEar methodology from Wei et al. (EACL 2026). The core technique constructs controlled perturbation sets across three axes — linguistic (language, accent), demographic (gender), and structural (answer option ordering) — then measures model sensitivity using four complementary metrics that together reveal whether a speech model treats all speakers and configurations fairly or amplifies hidden biases.

## When to Use

- When the user wants to evaluate whether an audio/speech LLM produces different answers based on speaker accent, gender, or language
- When building a fairness benchmark for a multimodal model that accepts speech input
- When the user asks to measure option-order sensitivity (positional bias) in any multiple-choice evaluation, text or audio
- When auditing a voice assistant or speech-to-text-to-LLM pipeline for demographic bias before deployment
- When comparing robustness between end-to-end audio models and ASR-then-LLM pipeline architectures
- When the user needs to quantify whether chain-of-thought prompting reduces bias in speech models
- When constructing a speech-augmented version of an existing text benchmark (e.g., converting MMLU to spoken form)

## Key Technique

**Controlled Perturbation Design.** The BiasInEar method takes a fixed set of questions and generates multiple spoken versions that differ along exactly one axis at a time. For a single question, you produce audio variants spanning genders (male/female), accents (e.g., American/British/Indian English), languages (English/Chinese/Korean), and option orderings (original A-B-C-D vs. reversed D-C-B-A). This yields up to 28 configurations per question. The key insight is that a fair model should produce the same answer regardless of which variant it hears — any divergence signals bias.

**Four-Metric Evaluation Framework.** No single metric captures all bias dimensions, so BiasInEar uses four in concert. *Accuracy* measures raw correctness per condition. *Shannon Entropy* (base-4 normalized to [0,1]) quantifies how uncertain the model is across answer options within a condition. *APES (Average Pairwise Entropy Shift)* measures the mean absolute entropy difference between all pairs of levels within a variable — low APES means high robustness, high APES means the model's confidence shifts depending on, say, accent. *Fleiss' kappa* measures categorical agreement across perturbations, correcting for chance — kappa near 1 means the model picks the same answer regardless of perturbation, kappa near 0 means it is effectively random across conditions.

**Critical Finding: Speech Amplifies Structural Bias.** The paper demonstrates that models robust to gender and accent perturbations can still be highly sensitive to language choice and option ordering. Option-order reversal alone causes 0.5%–6.75% accuracy drops. This means any serious audio model audit must test positional bias, not just demographic fairness. Chain-of-thought prompting improved Fleiss' kappa by 19–27%, and larger models consistently showed lower APES and higher kappa — both actionable levers for mitigation.

## Step-by-Step Workflow

1. **Define the evaluation corpus.** Select or build a multiple-choice question set. If adapting an existing benchmark (e.g., MMLU), filter to questions whose text can be meaningfully spoken aloud. Exclude questions that rely on visual diagrams or complex mathematical notation that cannot be verbalized.

2. **Rewrite questions for speech.** Use an LLM to convert symbolic notation, abbreviations, and formatting into natural spoken language. For example, rewrite `H₂O` as "H two O" and `≥ 5` as "greater than or equal to five". Diff the rewritten version against the original to catch semantic drift, and manually verify a stratified sample.

3. **Generate speech variants with controlled perturbations.** Using a multilingual TTS system (e.g., Gemini TTS, Azure Speech, or a high-quality open-source model), synthesize each question in every target configuration:
   - **Gender:** male and female voices
   - **Accent:** 2–3 regional accents per language (e.g., American/British/Indian for English)
   - **Language:** each target language
   - **Option order:** original (A-B-C-D) and reversed (D-C-B-A) at minimum; optionally add random shuffles
   Concatenate the question audio with each answer option audio to form complete audio prompts.

4. **Validate audio quality.** Run each clip through two independent ASR systems (e.g., Whisper Large v3 and an omnilingual ASR). Compute WER against the intended transcript, taking the minimum WER across systems. Flag clips above a threshold (e.g., WER > 0.3) for re-generation or exclusion. Manually audit a stratified sample of ~40 clips per language per WER bin.

5. **Run model inference.** Feed each audio variant to the target model(s) with temperature=0 for reproducibility. Use a standardized prompt instructing the model to select exactly one answer option (A/B/C/D). Run both standard prompting and chain-of-thought prompting to compare reasoning strategies.

6. **Parse and validate responses.** Extract the selected answer from each model response. Handle formatting inconsistencies (e.g., "The answer is B" vs. "B)" vs. "B"). Discard unparseable responses and report the discard rate.

7. **Compute the four metrics per variable.**
   - **Accuracy:** percentage of correct answers per condition level (e.g., per accent, per gender)
   - **Entropy:** `H_q = -sum over options o of p_q(o) * log4(p_q(o))`, where `p_q(o)` is the fraction of perturbation runs selecting option `o`. Compute within each level as `H_q^l`.
   - **APES:** `APES_q^v = (2 / L(L-1)) * sum over all pairs (l_i, l_j) of |H_q^l_i - H_q^l_j|`, where L is the number of levels for variable v. Average across all questions.
   - **Fleiss' kappa:** `kappa = (P_bar - P_e) / (1 - P_e)`, treating each perturbation as a rater and each question's answer as a category. Compute per variable.

8. **Analyze and compare.** Generate a summary table with one row per model (or model configuration) and columns for each metric broken down by variable (gender, accent, language, option order). Flag any variable where APES > 0.1 or kappa < 0.4 as a fairness concern. Compare standard vs. CoT prompting to quantify reasoning-based mitigation.

9. **Report findings with actionable recommendations.** For each flagged dimension, state the observed sensitivity magnitude (accuracy gap, APES value, kappa range) and suggest mitigations: use CoT prompting, scale up model size, switch from end-to-end to pipeline architecture, or augment training data along the sensitive axis.

10. **Export artifacts.** Save per-question per-condition raw predictions, computed metrics as CSV/JSON, and a summary report. Provide reproducibility metadata: model versions, TTS system, ASR systems used for validation, temperature setting, and prompt templates.

## Concrete Examples

**Example 1: Auditing a voice assistant for accent bias**

User: "I'm deploying a voice assistant that answers medical questions. I need to check if it gives different answers depending on the user's accent."

Approach:
1. Select 200 medical multiple-choice questions from a validated corpus (e.g., MedQA subset).
2. Rewrite questions for speech: convert "mg/dL" to "milligrams per deciliter", etc.
3. Synthesize each question in 3 accents (American, British, Indian English) x 2 genders = 6 audio variants per question.
4. Validate with Whisper ASR; exclude clips with WER > 0.25.
5. Run all 1,200 audio clips through the voice assistant at temperature=0.
6. Compute metrics per accent:

```
Output:
┌─────────────┬──────────┬─────────┬───────┬────────┐
│ Accent      │ Accuracy │ Entropy │ APES  │ Kappa  │
├─────────────┼──────────┼─────────┼───────┼────────┤
│ American    │ 78.5%    │ 0.12    │       │        │
│ British     │ 77.0%    │ 0.14    │ 0.031 │ 0.72   │
│ Indian      │ 71.5%    │ 0.22    │       │        │
├─────────────┼──────────┼─────────┼───────┼────────┤
│ Conclusion: Indian accent shows 7% accuracy drop  │
│ and elevated entropy. APES=0.031 is low (good),   │
│ but kappa=0.72 (substantial, not perfect).         │
│ Recommendation: augment training with more Indian  │
│ English speech data; test with CoT prompting.      │
└────────────────────────────────────────────────────┘
```

**Example 2: Measuring option-order bias in a speech-based exam system**

User: "We're building an AI-graded spoken exam. Does the order of answer choices affect the model's selection?"

Approach:
1. Take 500 exam questions in their original A-B-C-D order.
2. Generate a reversed version D-C-B-A for each (remap correct answer accordingly).
3. Synthesize both orderings in a single voice/accent/language configuration to isolate the positional variable.
4. Run both sets through the grading model.
5. Compute per-question agreement and aggregate metrics:

```
Output:
Option Order Sensitivity Analysis (N=500)
─────────────────────────────────────────
Accuracy (original order):  82.4%
Accuracy (reversed order):  76.8%
Accuracy gap:                5.6%  ← SIGNIFICANT

APES (option order):         0.087
Fleiss' kappa:               0.41  ← only moderate agreement

Per-position accuracy (original order):
  Correct answer at A: 86.2%
  Correct answer at B: 83.1%
  Correct answer at C: 80.4%
  Correct answer at D: 79.9%

Diagnosis: Strong primacy bias — model favors earlier positions.
Mitigation: (1) Use CoT prompting (expected +19-27% kappa improvement),
(2) Average predictions across 2+ random shuffles before final answer,
(3) Report shuffled-ensemble accuracy as the official score.
```

**Example 3: Comparing pipeline vs. end-to-end architectures for multilingual fairness**

User: "We're choosing between a Whisper+GPT pipeline and an end-to-end audio model. Which is fairer across languages?"

Approach:
1. Select 300 questions, generate audio in English, Chinese, and Korean with matched voices.
2. Run through both architectures with identical prompts.
3. Compute cross-language APES and kappa for each architecture:

```
Output:
Cross-Language Robustness Comparison
────────────────────────────────────
                     Pipeline (ASR→LLM)    End-to-End
                     ──────────────────    ──────────
Avg Accuracy (EN):   80.3%                 79.1%
Avg Accuracy (ZH):   74.7%                 68.2%
Avg Accuracy (KO):   72.1%                 63.5%

APES (language):     0.042                 0.098
Fleiss' kappa:       0.58                  0.31

Verdict: Pipeline architecture shows 2.3x lower APES and
substantially higher kappa across languages. End-to-end model
shows a 15.6% accuracy spread vs. pipeline's 8.2% spread.

Recommendation: For multilingual deployment, pipeline
architecture provides significantly more consistent behavior
across languages. If end-to-end is required, prioritize
models with explicit cross-lingual pretraining.
```

## Best Practices

- **Do:** Always test option-order sensitivity — it is consistently the largest source of bias in the BiasInEar findings, often exceeding demographic bias by a wide margin.
- **Do:** Use all four metrics together. Accuracy alone hides problems: a model can maintain similar accuracy across accents while dramatically shifting its confidence distribution (detected by entropy and APES).
- **Do:** Use temperature=0 and fixed random seeds for reproducibility. Bias audits must be deterministic to be credible.
- **Do:** Run both standard and chain-of-thought prompting variants. CoT improved Fleiss' kappa by 19–27% in the original study and is a low-cost mitigation.
- **Avoid:** Using a single ASR system for audio quality validation. Use at least two independent ASR systems and take the minimum WER to reduce false rejections from ASR-specific errors.
- **Avoid:** Testing only one perturbation axis. A model that is fair on gender can still be deeply biased on language or option order. Always test all three axes: demographic, linguistic, and structural.
- **Avoid:** Drawing conclusions from small sample sizes. APES and kappa require sufficient question counts per condition to be statistically meaningful — aim for at least 100 questions per configuration.

## Error Handling

- **TTS generation failures:** Some questions with unusual characters or code snippets will produce garbled audio. Pre-filter these during the rewriting step; verify with ASR-based WER screening. Re-generate failed clips up to 3 times before excluding.
- **Unparseable model responses:** Models may produce verbose answers instead of a clean A/B/C/D selection. Implement robust regex parsing (match patterns like "answer is B", "(B)", "B.", "B)"). Log and report the unparseable rate — if it exceeds 5% for any condition, the prompt template needs revision.
- **Degenerate entropy values:** If a model selects the same option for all variants of a question, entropy is 0 and APES is 0. This is not an error — it means the model is consistent (though possibly consistently wrong). Report accuracy alongside to distinguish consistent-correct from consistent-incorrect.
- **Insufficient perturbation levels:** Fleiss' kappa requires at least 2 raters (perturbation variants). If you have only 2 levels for a variable (e.g., male/female), kappa reduces to Cohen's kappa. This is valid but note the reduced statistical power.
- **Cross-language answer mapping errors:** When reversing option order across languages, ensure the correct-answer label is remapped correctly (e.g., if correct answer was A in original, it becomes D in reversed). A single mapping bug invalidates the entire positional analysis.

## Limitations

- The framework is designed for multiple-choice evaluation. Open-ended generation tasks require different bias measurement approaches (e.g., embedding-based semantic similarity across conditions).
- TTS-generated speech does not perfectly represent natural human speech. Real-world accents have far more variation than TTS can produce. Results indicate a ceiling on ecological validity.
- The four metrics assume discrete answer categories. For models that output continuous scores or rankings, adaptations are needed (e.g., replacing Shannon entropy with differential entropy).
- Language coverage in the original study is limited to English, Chinese, and Korean. Extending to morphologically rich or low-resource languages may require additional question-rewriting heuristics and TTS quality validation.
- The method detects bias but does not explain its cause. A high APES for accent may stem from ASR-layer errors, attention-layer sensitivity, or training data imbalance — further ablation is needed to diagnose root causes.

## Reference

Wei, S.-L., Liao, Y.-L., Chang, Y.-H., Huang, H.-H., & Chen, H.-H. (2026). *Bias in the Ear of the Listener: Assessing Sensitivity in Audio Language Models Across Linguistic, Demographic, and Positional Variations.* EACL 2026 Findings. [arXiv:2602.01030](https://arxiv.org/abs/2602.01030v1). Look for: the APES metric definition (Section 4.2), the option-order reversal methodology (Section 5.3), and Table 8 comparing text vs. speech bias amplification.