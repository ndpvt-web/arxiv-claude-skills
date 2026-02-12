---
name: "creditaudit-2textnd-dimension-evaluation"
description: "Evaluate and select LLMs using CreditAudit's 2D framework: mean ability plus stability risk (fluctuation) across system prompt variations. Assigns credit grades (AAA–BBB) to models based on performance volatility. Use when: 'compare models for deployment', 'which LLM is most stable', 'evaluate model robustness to prompt changes', 'credit grade these models', 'model selection for agentic pipeline', 'rank models by reliability'."
---

CreditAudit enables Claude to evaluate and compare language models not just by average benchmark scores, but by a second critical dimension: **stability under routine system prompt variation**. Based on the paper "CreditAudit: 2nd Dimension for LLM Evaluation and Selection" (arXiv:2602.02515v2), this skill implements a deployment-oriented credit audit framework that tests models across semantically aligned prompt templates, computes mean ability (mu) and scenario-induced fluctuation (sigma), and maps volatility into interpretable credit grades from AAA (most stable) to BBB (most volatile). This directly addresses the real-world problem where leaderboard-similar models behave very differently when system prompts, output protocols, or interaction modes shift during production use.

## When to Use

- When the user asks to compare multiple LLMs for a production deployment decision and needs more than just accuracy scores
- When evaluating model robustness for agentic or multi-step pipelines where small prompt shifts can cascade into failures
- When the user wants to understand which model is most stable across different system prompt formulations
- When building a model selection matrix for tiered deployment (e.g., safety-critical vs. cost-optimized tiers)
- When the user says "which model should I deploy" or "compare these models" and wants a principled framework
- When assessing whether a model's high benchmark score is reliable or fragile under routine prompt iteration
- When the user needs to justify a model choice to stakeholders with a credit-grade-style risk rating

## Key Technique

CreditAudit treats model evaluation as a **2D problem**: the X-axis is mean ability (mu) — average performance across prompt scenarios — and the Y-axis is fluctuation (sigma) — standard deviation of performance across those same scenarios. Two models with identical mean scores can land in entirely different risk categories if one is stable (sigma=0.5) while the other swings wildly (sigma=3.0). This matters because in agentic pipelines, a model that occasionally fails badly under minor prompt rewording will cause compounding downstream failures.

The framework constructs a **family of semantically aligned, non-adversarial system prompt templates** (typically 8–10 variants) that represent routine protocol variations practitioners actually encounter: "output only the option letter," "be concise," "think step by step," "be cautious," format-constrained variants, etc. These are not adversarial jailbreaks — they are the mundane prompt rewrites that happen during normal iteration. Each model is evaluated on the same question set under every template, producing a model x template x benchmark score cube.

Fluctuation sigma is then mapped to **credit grades** using cross-model quantile thresholds: AAA (sigma <= q0.25, most stable), AA (q0.25 < sigma <= q0.50), A (q0.50 < sigma <= q0.75), and BBB (sigma > q0.75, most volatile). A scenario neutrality diagnostic confirms that templates don't introduce systematic difficulty bias — the observed fluctuation reflects genuine model-specific sensitivity. Selection then follows regime-specific rules: for agentic/high-failure-cost settings, prioritize low sigma first; for single-shot controlled deployments, score can weigh more heavily.

## Step-by-Step Workflow

1. **Define the evaluation task set.** Select or sample a fixed set of questions from one or more benchmarks relevant to the deployment (e.g., domain-specific QA, coding tasks, reasoning problems). Use a fixed random seed for reproducibility. Aim for 100–500 questions per benchmark.

2. **Construct 8–10 semantically aligned system prompt templates.** Each template should express a different but realistic protocol intent: bare-minimum instruction, concise output, verbose reasoning, structured JSON output, cautious hedging, step-by-step chain-of-thought, role-play framing, format-constrained (e.g., "answer with only A/B/C/D"), etc. Crucially, the same template index must express the same intent across all benchmarks.

3. **Run each model against every (template, question) pair.** For M models, T templates, and N questions, this produces M x T x N raw responses. Extract the answer from each response and compute accuracy per (model, template, benchmark) cell.

4. **Compute per-model aggregate scores.** For each model m and template t, compute the equal-weight average score across benchmarks: S(m,t). Then compute mean ability: `mu_m = (1/T) * sum(S(m,t) for t in templates)` and fluctuation: `sigma_m = sqrt((1/(T-1)) * sum((S(m,t) - mu_m)^2 for t in templates))`.

5. **Run the scenario neutrality diagnostic.** For each template t, compute the cross-model average: `S_bar_t = (1/M) * sum(S(m,t) for m in models)`. Verify the trend across templates is near-flat. If one template is dramatically harder/easier for ALL models, it's introducing difficulty drift rather than measuring model sensitivity — consider removing or rebalancing it.

6. **Assign credit grades using cross-model quantiles.** Compute q0.25, q0.50, q0.75 of sigma across all evaluated models. Map each model: AAA if sigma <= q0.25, AA if sigma <= q0.50, A if sigma <= q0.75, BBB otherwise.

7. **Plot the 2D evaluation map.** Place models on a (mu, sigma) plane. Identify four quadrants: Q1 (high score, low sigma) = safe default; Q2 (lower score, low sigma) = predictable baseline; Q3 (lower score, high sigma) = avoid; Q4 (high score, high sigma) = scenario-fragile, use with caution.

8. **Apply regime-specific selection rules.** For agentic/multi-step pipelines: filter to AAA/AA grades first, then rank by mu within that tier. For single-shot controlled deployments: rank by mu but flag Q4 models with a stability warning. For cost-sensitive tiers: Q2 models offer predictable behavior at lower capability.

9. **Generate the CreditAudit report.** Produce a structured summary: model rankings table with mu, sigma, grade, and quadrant; per-benchmark breakdowns of mu and sigma; the scenario neutrality check results; and regime-specific deployment recommendations.

10. **Iterate templates if needed.** If sigma values cluster too tightly (all models grade AAA), templates may be too similar — add more diverse protocol variations. If all models grade BBB, templates may be too aggressive — verify they're non-adversarial.

## Concrete Examples

**Example 1: Comparing models for an agentic coding pipeline**

User: "I'm choosing between GPT-4o, Claude Sonnet, and Gemini Pro for a multi-step code generation pipeline. Which is most reliable?"

Approach:
1. Define task set: 200 coding questions from HumanEval+ and MBPP
2. Construct 8 system prompt templates:
   - Temp00: "You are a helpful coding assistant."
   - Temp01: "Output only the function body, no explanation."
   - Temp02: "Think step by step, then write the code."
   - Temp03: "You are a senior software engineer. Write production-quality code."
   - Temp04: "Be concise. Return only working code."
   - Temp05: "Write the solution in a markdown code block with the language specified."
   - Temp06: "First analyze the problem, then implement. Include type hints."
   - Temp07: "You are cautious and thorough. Handle edge cases."
3. Run all 3 models x 8 templates x 200 questions
4. Compute mu and sigma per model

Output:
```
CreditAudit Report — Agentic Coding Pipeline Selection
═══════════════════════════════════════════════════════

Model           | mu (%)  | sigma | Grade | Quadrant
----------------|---------|-------|-------|---------
Claude Sonnet   |  82.3   |  1.1  |  AAA  |   Q1
GPT-4o          |  83.1   |  2.8  |   A   |   Q4
Gemini Pro      |  79.5   |  1.4  |  AA   |   Q2

Quantile thresholds: q25=1.2  q50=1.6  q75=2.4

Scenario Neutrality: PASS (cross-model template means range: 79.1–82.4%)

Regime Recommendation (Agentic/Multi-Step):
  → Deploy Claude Sonnet (AAA grade, Q1). Despite GPT-4o's +0.8% mean
    advantage, its sigma of 2.8 means prompt rewording during iteration
    risks disproportionate failures in downstream pipeline steps.
  → Gemini Pro is a viable fallback (AA, predictable baseline).
  → GPT-4o requires prompt-locking if deployed — avoid routine prompt
    iteration without re-testing.
```

**Example 2: Single-shot QA deployment with stability audit**

User: "Rank these 5 models for a customer-facing Q&A bot. Accuracy matters most but I need to know which ones are flaky."

Approach:
1. Define task set: 300 questions from TruthfulQA + domain-specific QA
2. Construct 10 templates varying tone (formal, casual), format (brief, detailed), and instruction style (direct, role-play)
3. Run all 5 models x 10 templates x 300 questions
4. Compute mu, sigma, grades

Output:
```
CreditAudit Report — Customer QA Bot Selection
═══════════════════════════════════════════════

Model      | mu (%) | sigma | Grade | Quadrant
-----------|--------|-------|-------|---------
Model A    |  88.2  |  0.9  |  AAA  |   Q1
Model B    |  87.9  |  1.1  |  AAA  |   Q1
Model C    |  89.1  |  3.2  |  BBB  |   Q4  ⚠
Model D    |  84.3  |  1.5  |  AA   |   Q2
Model E    |  83.0  |  2.7  |   A   |   Q3

Regime Recommendation (Single-Shot, Customer-Facing):
  → Model A or B (both AAA, Q1) are safe defaults.
  → Model C scores highest BUT grade BBB — its 89.1% mean hides
    template-specific drops to 83.4%. If your system prompt changes
    during A/B testing or iteration, expect inconsistent behavior.
  → Stability warning for Model C: per-benchmark sigma on TruthfulQA
    is 4.1 — truthfulness degrades significantly under casual-tone
    templates.
```

**Example 3: Designing a CreditAudit template family from scratch**

User: "Help me create a set of system prompt templates for evaluating models on my legal document analysis task."

Approach:
1. Identify routine protocol variations in legal NLP deployments
2. Construct semantically aligned templates

Output:
```
CreditAudit Template Family — Legal Document Analysis
══════════════════════════════════════════════════════

Temp00 (Baseline):
  "You are a legal document analysis assistant. Answer the question
   based on the provided document."

Temp01 (Format-constrained):
  "Answer with only the relevant legal citation and a one-sentence
   explanation. No preamble."

Temp02 (Chain-of-thought):
  "Analyze the legal document step by step. Identify relevant clauses,
   then provide your answer with reasoning."

Temp03 (Cautious):
  "You are a careful legal analyst. If the answer is ambiguous, say so.
   Err on the side of caution."

Temp04 (Structured output):
  "Return your answer as JSON with keys: 'answer', 'confidence',
   'relevant_clauses', 'reasoning'."

Temp05 (Role-play):
  "You are a senior associate at a law firm reviewing this document
   for a partner. Be thorough and precise."

Temp06 (Concise):
  "Be brief. Provide only the direct answer to the question."

Temp07 (Adversarial-adjacent but non-adversarial):
  "The user may phrase questions ambiguously. Interpret charitably
   and answer the most likely intended question."

Temp08 (Multi-part):
  "First state whether the question can be answered from the document.
   If yes, provide the answer. If partially, state what's missing."

Alignment check: Each template varies protocol intent (format, tone,
reasoning depth, output structure) while keeping the core task identical.
All templates are non-adversarial — they represent variations a legal
tech team would plausibly iterate through during development.
```

## Best Practices

- **Do:** Keep templates non-adversarial and semantically aligned. They should represent prompt variations that actually occur during normal development iteration — not jailbreaks or intentionally confusing instructions.
- **Do:** Use the same fixed question set across all templates and models. Template-induced variance is only meaningful when the task is held constant.
- **Do:** Always run the scenario neutrality diagnostic before assigning grades. If one template is universally harder, it measures task difficulty, not model stability.
- **Do:** Report both mu AND sigma together. A model's credit grade without its mean ability is incomplete — AAA at 60% accuracy is not better than BBB at 95% for all use cases.
- **Avoid:** Using fewer than 6 templates. With too few data points, sigma estimates are unreliable and grades become noisy.
- **Avoid:** Mixing adversarial and non-adversarial templates in the same audit. Adversarial robustness is a different evaluation axis — CreditAudit specifically measures sensitivity to routine, benign prompt variation.
- **Avoid:** Setting absolute sigma thresholds (e.g., "sigma > 2 is bad"). The quantile-based grading adapts to the model cohort — what matters is relative stability within the candidate set.
- **Avoid:** Ignoring per-benchmark sigma breakdowns. A model may be AAA overall but BBB on truthfulness specifically — regime-specific decisions require granular data.

## Error Handling

- **All models cluster at the same grade:** Your templates lack sufficient diversity. Add templates with different output format constraints, reasoning depth requirements, or role framings. Verify that templates actually produce different model behaviors by inspecting raw response patterns.
- **Scenario neutrality check fails:** One or more templates are systematically harder or easier for all models. Inspect the cross-model template means. Remove or replace the outlier template, or apply a difficulty-centering adjustment: subtract the template mean from each score before computing sigma.
- **Sigma is dominated by one benchmark:** Decompose into per-benchmark mu and sigma. Report benchmark-specific grades alongside the aggregate. The deployment decision should weight the benchmark most relevant to the target use case.
- **Sample size too small for reliable sigma:** With fewer than 50 questions per benchmark, accuracy estimates per template are noisy, inflating sigma. Increase the evaluation set or use confidence intervals around sigma estimates.
- **Model API non-determinism inflates sigma:** Run each (model, template, question) pair multiple times with temperature=0. If variance persists, separate API-induced noise from template-induced fluctuation by computing within-template variance first.

## Limitations

- CreditAudit measures sensitivity to **non-adversarial** prompt variation only. It does not assess adversarial robustness, jailbreak resistance, or safety under attack.
- The credit grade is **relative to the evaluated cohort**. Adding or removing models from the comparison shifts quantile thresholds and can change grades. Always report the cohort composition.
- Template design requires domain expertise. Poorly chosen templates (too similar, or inadvertently adversarial) invalidate the stability signal.
- The framework assumes equal weighting across benchmarks. If one benchmark matters more for your deployment, apply explicit weights to the score aggregation before computing mu and sigma.
- CreditAudit does not measure cost, latency, or context window — it is purely an accuracy-stability framework. Combine with operational metrics for full deployment decisions.
- With only 4 grade levels (AAA–BBB), the framework provides coarse discrimination. For large model cohorts (20+), consider finer quantile bins or reporting raw sigma alongside grades.

## Reference

**Paper:** "CreditAudit: 2nd Dimension for LLM Evaluation and Selection" — Song et al., 2026. arXiv:2602.02515v2. https://arxiv.org/abs/2602.02515v2

Look for: The model x template x benchmark score cube methodology, the quantile-based grade mapping table, the scenario neutrality diagnostic, and the four-quadrant (mu, sigma) selection framework with regime-specific deployment guidance.