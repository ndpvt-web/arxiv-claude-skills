---
name: "can-reasoning-be-trusted"
description: "Validate and score LLM-generated statistical reasoning using a three-axis rubric (Correctness 40%, Explanation 35%, Reasoning 25%) and LLM-as-judge evaluation, based on Nagarkar et al. 2026. Use when: 'evaluate this statistical analysis', 'score this model output', 'check my stats reasoning', 'grade this explanation', 'build a stats evaluation pipeline', 'assess reasoning quality'."
---

# Statistical Reasoning Validation with LLM-as-Judge Evaluation

This skill enables Claude to evaluate, score, and validate statistical reasoning — whether produced by humans, LLMs, or automated pipelines — using the three-axis weighted rubric and LLM-as-judge framework from Nagarkar, Bogachev & Sharoff (2026). The core insight: LLM judges using domain-specific rubrics align with human expert judgment far better than traditional metrics (BLEU, BERTScore, perplexity), achieving MAE of 0.645 vs 1.073–3.092 for automated metrics on a 0–5 scale. This makes LLM-based evaluation a practical, scalable replacement for human grading of statistical work.

## When to Use

- When a user asks you to evaluate or grade a statistical solution, proof, or analysis
- When building automated assessment systems for statistics courses or data science training
- When validating the correctness of statistical reasoning in research papers or reports
- When implementing quality control for data analysis pipelines that produce statistical interpretations
- When comparing multiple model outputs on statistical tasks and need a principled scoring method
- When a user wants to check whether a hypothesis test, regression interpretation, or probability calculation is correctly reasoned
- When designing evaluation harnesses for fine-tuned models on quantitative tasks

## Key Technique

**Three-Axis Weighted Rubric.** The paper demonstrates that evaluating statistical reasoning requires decomposing quality into three distinct axes: Correctness (40% weight) — whether the numerical answer and statistical conclusion are right; Step-by-step Explanation (35%) — whether the derivation is logically structured and complete; and Interpretation & Reasoning (25%) — whether the solution demonstrates conceptual clarity and valid statistical justification. Each axis is scored on a 0–5 scale with half-point increments. This decomposition matters because a solution can be numerically correct but poorly reasoned, or well-explained but wrong — collapsing these into a single score hides actionable feedback.

**LLM-as-Judge Superiority.** Traditional text-similarity metrics fail catastrophically on statistical reasoning. BLEU scores had MAE of 3.092 against human judgment (on a 0–5 scale) and Kendall's tau of just 0.098 — essentially no correlation. BERTScore was better (MAE 1.073) but still showed near-zero rank correlation (tau 0.022). In contrast, a fine-tuned Mistral model acting as judge achieved MAE 0.645 and tau 0.482, with a Wilcoxon p-value of 0.076 (non-significant difference from human judges). The key: the judge model must be instructed with a domain-specific persona ("You are a Professor of Statistics") and the explicit rubric, not just asked to "rate this answer."

**Architecture-Dependent Fine-Tuning.** Not all models benefit equally from statistical fine-tuning. Mistral 7B showed +34.18% weighted score improvement after LoRA fine-tuning, while LLaMA-3 8B gained only +14.02%. This means model selection for judge roles matters — and evaluation pipelines should test multiple judge architectures rather than assuming one-size-fits-all.

## Step-by-Step Workflow

1. **Classify the statistical task by difficulty and domain.** Identify whether the problem involves probability, hypothesis testing, regression analysis, time series, ANOVA, or Bayesian statistics. Determine difficulty: basic (definitional, single-step), intermediate (multi-step computation), or advanced (requires modeling choices, proof, or interpretation of complex output).

2. **Produce or collect the candidate solution.** If generating the answer, solve the statistical problem step-by-step, showing all work. If evaluating an existing answer, collect the full response including any numerical results, derivation steps, and interpretive conclusions.

3. **Establish the reference solution.** Construct or retrieve the benchmark answer with correct numerical results, the canonical derivation path, and the expected statistical interpretation. This is the ground truth against which the candidate is scored.

4. **Score Correctness (weight: 40%).** Compare the candidate's numerical answer and statistical conclusions against the reference. Award 5 for exact match with correct conclusions, 3–4 for minor computational errors that don't affect the conclusion, 1–2 for fundamental errors in method or conclusion, 0 for completely wrong or unrelated answers.

5. **Score Explanation (weight: 35%).** Evaluate the logical structure and completeness of the derivation. Check: Are assumptions stated? Are formulas identified before application? Are intermediate steps shown? Is the flow from premise to conclusion traceable? Award 5 for complete, well-structured derivation; deduct for missing steps, unexplained jumps, or disorganized presentation.

6. **Score Reasoning (weight: 25%).** Assess conceptual understanding and statistical justification. Does the solution explain *why* a particular test or method is appropriate? Does it interpret results in context (e.g., what a p-value means for the specific hypothesis)? Does it acknowledge limitations or assumptions? Award 5 for deep conceptual clarity; deduct for mechanical application without understanding.

7. **Compute the weighted score.** Calculate: `weighted = (correctness * 0.40) + (explanation * 0.35) + (reasoning * 0.25)`. Map to a qualitative band: 4.5–5.0 Excellent, 3.5–4.0 Good, 2.5–3.0 Fair, 1.5–2.0 Poor, 0–1.0 Very Poor.

8. **Generate actionable feedback.** For each axis scoring below 4.0, produce specific remediation: identify the exact step where correctness failed, the missing derivation link, or the conceptual gap in reasoning. Do not just say "needs improvement" — point to the precise deficiency.

9. **Aggregate across multiple judges (if applicable).** When using multiple evaluators, compute arithmetic mean per axis and check inter-judge consistency. If the standard deviation across judges exceeds 1.0 on any axis, flag that item for manual review — high disagreement indicates an ambiguous or edge-case problem.

10. **Report with confidence calibration.** Present the final score alongside a confidence indicator. High confidence: all three axes agree within 0.5 points of the mean. Medium: one axis diverges by 0.5–1.5. Low: multiple axes show large spread or the problem is outside well-covered domains (e.g., novel Bayesian applications).

## Concrete Examples

**Example 1: Evaluating a Hypothesis Test Solution**

User: "Grade this student's answer — A factory claims their bolts have mean diameter 10mm. A sample of 30 bolts has mean 10.3mm and std dev 0.8mm. The student concludes: 'We use a z-test. z = (10.3-10)/(0.8/sqrt(30)) = 2.054. Since 2.054 > 1.96, we reject H0 at alpha=0.05. The bolts are defective.'"

Approach:
1. Reference: t-test is appropriate (sigma unknown, n=30), not z-test. t = 2.054, df=29, critical value ~2.045. Rejection is correct but for wrong reasons.
2. Correctness (score: 2.5/5): Numerical calculation is right, conclusion happens to be correct, but wrong test chosen (z vs t). The error is forgivable at n=30 but methodologically wrong.
3. Explanation (score: 3.0/5): Steps are shown clearly, formula stated, computation traceable. Missing: no statement of hypotheses (H0, H1), no mention of assumptions (normality, independence).
4. Reasoning (score: 1.5/5): "The bolts are defective" overstates the conclusion — rejection of H0 means evidence the mean differs from 10mm, not that bolts are defective. No discussion of practical significance vs statistical significance.
5. Weighted: (2.5 * 0.40) + (3.0 * 0.35) + (1.5 * 0.25) = 1.0 + 1.05 + 0.375 = 2.425 (Fair)

Output:
```
Score: 2.4/5.0 (Fair)
  Correctness:  2.5/5 — z-test used instead of t-test (unknown population σ).
                         Numerical result is coincidentally close but method is wrong.
  Explanation:  3.0/5 — Computation steps shown, but missing formal hypothesis
                         statement (H0: μ=10, H1: μ≠10) and assumption checks.
  Reasoning:    1.5/5 — "Bolts are defective" is an overstatement. Rejecting H0
                         only indicates the mean likely differs from 10mm.
                         No discussion of practical vs statistical significance.

Remediation:
- Use t-test when population σ is unknown, even at n=30.
- Always state H0 and H1 explicitly before computing.
- Distinguish "statistically significant difference" from "defective product."
```

**Example 2: Building an Automated Grading Pipeline**

User: "I want to build a system that automatically grades statistics homework. How should I set up the evaluation?"

Approach:
1. Design the evaluation schema using the three-axis rubric.
2. Implement the judge prompt with the Professor of Statistics persona.
3. Set up multi-judge aggregation for reliability.

Output:
```python
# evaluation_config.py — Statistics Homework Auto-Grader

JUDGE_SYSTEM_PROMPT = """You are a Professor of Statistics evaluating student solutions.
For each solution, score on three axes using the 0-5 scale (half-point increments):

1. CORRECTNESS (40%): Is the numerical answer correct? Is the statistical
   conclusion valid? Compare against the reference solution.
   5=exact match, 3-4=minor errors, 1-2=fundamental errors, 0=wrong/unrelated.

2. EXPLANATION (35%): Is the derivation logically structured and complete?
   Are assumptions stated? Are formulas identified? Are steps traceable?
   5=complete and clear, 3-4=mostly complete, 1-2=major gaps, 0=absent.

3. REASONING (25%): Does the student understand WHY this method applies?
   Are results interpreted in context? Are limitations acknowledged?
   5=deep understanding, 3-4=adequate, 1-2=mechanical, 0=no understanding.

Output format:
CORRECTNESS: [score] — [one-line justification]
EXPLANATION: [score] — [one-line justification]
REASONING: [score] — [one-line justification]
WEIGHTED: [computed score]
FEEDBACK: [specific remediation for any axis below 4.0]
"""

RUBRIC_WEIGHTS = {"correctness": 0.40, "explanation": 0.35, "reasoning": 0.25}

QUALITY_BANDS = {
    (4.5, 5.0): "Excellent",
    (3.5, 4.5): "Good",
    (2.5, 3.5): "Fair",
    (1.5, 2.5): "Poor",
    (0.0, 1.5): "Very Poor",
}

def compute_weighted_score(correctness, explanation, reasoning):
    return (correctness * 0.40) + (explanation * 0.35) + (reasoning * 0.25)

def aggregate_judges(scores_list):
    """Average across multiple judge scores, flag high-disagreement items."""
    import statistics
    axes = ["correctness", "explanation", "reasoning"]
    aggregated = {}
    flags = []
    for axis in axes:
        values = [s[axis] for s in scores_list]
        aggregated[axis] = statistics.mean(values)
        if statistics.stdev(values) > 1.0:
            flags.append(f"High disagreement on {axis} (stdev={statistics.stdev(values):.2f})")
    aggregated["weighted"] = compute_weighted_score(**{a: aggregated[a] for a in axes})
    return aggregated, flags
```

**Example 3: Validating a Regression Interpretation**

User: "Check this interpretation: 'Our linear regression of house prices on square footage gives R-squared=0.72 and coefficient=150.3 (p<0.001). This proves that adding one square foot increases house value by exactly $150.30.'"

Approach:
1. Identify the statistical claims: R-squared interpretation, coefficient interpretation, causal language.
2. Score each axis against correct statistical reasoning.

Output:
```
Score: 2.1/5.0 (Poor)
  Correctness:  3.0/5 — R-squared and coefficient values reported correctly.
                         But "proves" and "exactly" are statistically incorrect claims.
  Explanation:  2.0/5 — No mention of confidence interval for the coefficient,
                         no residual diagnostics, no model assumptions stated.
  Reasoning:    1.0/5 — Three critical errors: (1) "proves" — regression shows
                         association, not causation without experimental design;
                         (2) "exactly $150.30" — the coefficient is an estimate
                         with uncertainty (report the CI); (3) R-squared=0.72
                         means 72% of variance explained, not mentioned.

Remediation:
- Replace "proves" with "is associated with" unless causal design justifies it.
- Report the 95% CI for the coefficient (e.g., "$150.30 [CI: $142.10–$158.50]").
- State R-squared interpretation: "72% of variance in price is explained by sq ft."
- Discuss residual plots and linearity assumption before interpreting coefficients.
```

## Best Practices

**Do:**
- Always decompose evaluation into the three separate axes before computing a weighted score. Collapsing into a single holistic rating loses the diagnostic value that makes this framework actionable.
- Use the "Professor of Statistics" persona in judge prompts — domain-specific framing measurably improves alignment with human expert judgment (MAE drops from ~1.1 to ~0.6).
- Require half-point scoring increments (0, 0.5, 1.0, ..., 5.0) to force discrimination between close answers while avoiding false precision.
- Provide the reference/benchmark solution to the judge alongside the candidate answer. Without a reference, correctness scoring becomes unreliable.

**Avoid:**
- Do not use BLEU, ROUGE, or BERTScore as primary metrics for statistical reasoning quality. These metrics have near-zero correlation with human judgment on reasoning tasks (Kendall's tau < 0.1).
- Do not assume a single judge is sufficient for high-stakes evaluation. Use at least two independent judge passes and flag items where scores diverge by more than 1.0 on any axis.
- Do not conflate "correct final answer" with "good reasoning." A solution that arrives at the right number through flawed logic should score high on correctness but low on reasoning — the rubric exists to capture exactly this distinction.
- Do not apply this rubric to purely computational problems with no reasoning component (e.g., "compute 2+2"). The framework adds value specifically when explanation and interpretation matter.

## Error Handling

- **Judge refuses to score or gives uniform scores:** The problem may be outside the judge's domain knowledge. Fall back to axis-by-axis manual review, or break the problem into sub-parts the judge can handle individually.
- **Scores cluster at extremes (all 5s or all 0s):** The reference solution may be poorly specified, or the rubric prompt is not being followed. Re-check that the system prompt includes the full 0–5 scale with descriptions and the explicit three-axis structure.
- **High inter-judge disagreement (stdev > 1.0):** The problem is ambiguous or admits multiple valid approaches. Add a fourth judge and weight by majority consensus, or flag for human adjudication. Common in advanced Bayesian problems where prior choice is subjective.
- **Candidate answer is much longer/shorter than reference:** Length differences can bias holistic scoring. The three-axis rubric mitigates this — a concise but correct answer should score high on correctness and reasoning even if explanation is sparse.
- **Non-statistical content mixed in:** If the answer includes irrelevant material (e.g., code output, data loading), instruct the judge to evaluate only the statistical reasoning portions and ignore boilerplate.

## Limitations

- The rubric is calibrated for classical statistics (probability, hypothesis testing, regression, time series, ANOVA, Bayesian fundamentals). For specialized domains like survival analysis, causal inference, or machine learning theory, the three-axis weights may need adjustment.
- LLM judges, even with domain-specific prompts, still show MAE ~0.6 against human experts. For pass/fail decisions with real consequences (academic grading, publication review), use the LLM judge as a first-pass filter, not the final arbiter.
- The framework assumes a reference solution exists. For open-ended exploratory data analysis where multiple valid approaches exist, correctness scoring becomes subjective and the 40% weight on correctness may over-penalize creative but valid alternatives.
- Fine-tuning results in the paper used 7–8B parameter models. The rubric and judge framework transfer to larger models without fine-tuning, but the specific performance numbers (e.g., +34% improvement) are architecture-specific and should not be cited as general expectations.

## Reference

Nagarkar, C., Bogachev, L., & Sharoff, S. (2026). *Can LLM Reasoning Be Trusted? A Comparative Study: Using Human Benchmarking on Statistical Tasks.* arXiv:2601.14479v1. Key sections: Table 2 (human scoring results), Table 3 (metric reliability comparison), Section 4 (LLM-as-judge rubric and persona design). Dataset: https://github.com/crishnagarkarleeds/statistics-llm