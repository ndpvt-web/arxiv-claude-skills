---
name: "creditaudit-2textnd-dimension-evaluation"
description: "Evaluate and select LLMs using 2D credit audit scoring (mean ability + stability risk). Computes performance mean (mu) and scenario-induced fluctuation (sigma) across system prompt variants, then assigns credit grades AAA-BBB via cross-model quantiles. Triggers: 'which model should I deploy', 'compare LLM stability', 'credit audit models', 'evaluate model robustness to prompt variation', 'model selection for agentic pipeline', 'rank models by reliability'"
---

CreditAudit enables Claude to perform deployment-oriented LLM evaluation that goes beyond single-score leaderboard comparisons. Instead of ranking models solely by accuracy, this skill applies a 2D framework: **mean ability** (average accuracy across prompt scenarios) and **scenario-induced fluctuation** (standard deviation across those scenarios), then maps fluctuation into interpretable credit grades (AAA through BBB). This directly addresses the real-world problem where models with near-identical benchmark scores behave very differently when system prompts, output formats, or interaction modes change during deployment.

## When to Use

- When the user asks which LLM to deploy for a production application, especially agentic or multi-step pipelines
- When comparing two or more models that score similarly on benchmarks but need differentiation for reliability
- When the user wants to evaluate model robustness to system prompt variations (format instructions, tone directives, output constraints)
- When designing a model evaluation harness that accounts for stability, not just peak performance
- When the user needs to justify model selection decisions with a structured risk framework (e.g., for compliance or engineering review)
- When building CI/CD pipelines that test LLM behavior across prompt template variants before deployment

## Key Technique

Standard LLM evaluation reports a single accuracy number per benchmark. CreditAudit adds a second dimension: **how much does that accuracy fluctuate when the system prompt changes in routine, non-adversarial ways?** The insight is that two models scoring 65% on MMLU may behave very differently in production -- one might hold steady at 63-67% across prompt variants while the other swings from 58% to 72%. The volatile model is a deployment liability, especially in agentic workflows where a single format violation or refusal can cascade into pipeline failure.

The framework constructs a family of **semantically aligned, non-adversarial system prompt templates** -- representing normal deployment variations like "output only the option letter," "be concise," "follow strict format," "verify before answering." Each model is evaluated on every template across fixed benchmark subsets. From the resulting score matrix, two statistics are computed per model: **mu** (mean accuracy across all templates) and **sigma** (sample standard deviation across templates). Sigma captures scenario-induced fluctuation -- how sensitive the model is to routine prompt changes.

Credit grades are then assigned by ranking all evaluated models by sigma and cutting at cross-model quantiles: **AAA** (sigma <= 25th percentile, most stable), **AA** (25th-50th), **A** (50th-75th), **BBB** (above 75th, most volatile). A neutrality diagnostic checks that templates do not introduce systematic difficulty drift by verifying that cross-model mean scores remain approximately flat across templates. The 2D plane (mu vs. sigma) creates four deployment quadrants: Q1 (high score, low sigma) is the safe default; Q4 (high score, high sigma) is strong but fragile; Q2 (low score, low sigma) is a predictable baseline; Q3 (low score, high sigma) should be avoided.

## Step-by-Step Workflow

1. **Define the model set.** List all candidate models to evaluate. CreditAudit uses cross-model quantiles, so include at least 4-6 models to produce meaningful grade boundaries. Include both frontrunners and baselines.

2. **Design 5-10 semantically aligned system prompt templates.** Each template should represent a realistic deployment variation -- not adversarial jailbreaks, but normal protocol changes. Examples: "Answer with only the letter of the correct option," "Think step by step, then give your final answer on a new line," "Be concise and direct," "Output your answer in JSON format with a 'choice' field," "Verify your reasoning before committing to an answer." Ensure templates are benchmark-agnostic or create aligned variants per benchmark.

3. **Select evaluation benchmarks and fix subsets.** Choose 2-4 benchmarks covering target capabilities (e.g., reasoning, knowledge, truthfulness). Sample a fixed subset of items per benchmark (200-500 items) and lock it. All models see the exact same items to ensure horizontal comparability. Record the seed used for sampling.

4. **Run the evaluation matrix.** For each (model, template, benchmark, item) tuple, generate the model's response with the template as the system prompt. Parse the response to extract the model's answer using a deterministic parser `g()`. Score correctness as a binary indicator against ground truth.

5. **Compute benchmark-level and overall scores.** For each model `m` and template `t`: compute per-benchmark accuracy `S_m,t,b = mean(correct predictions)`, then compute overall score `S_m,t = mean(S_m,t,b across benchmarks)` using equal weighting.

6. **Compute mu and sigma per model.** Mean ability: `mu_m = mean(S_m,t across all T templates)`. Fluctuation: `sigma_m = sample_std(S_m,t across all T templates)`. Also compute per-benchmark mu and sigma for diagnostic breakdowns.

7. **Run the neutrality diagnostic.** For each template `t`, compute `S_bar_t = mean(S_m,t across all models)`. Plot or inspect these values. If `S_bar_t` varies substantially across templates, some templates are systematically harder -- revise or remove those templates and re-run. The signal should come from model-specific sensitivity, not template difficulty.

8. **Assign credit grades via cross-model quantiles.** Compute q25, q50, q75 of sigma values across all models. Grade each model: AAA if `sigma_m <= q25`, AA if `q25 < sigma_m <= q50`, A if `q50 < sigma_m <= q75`, BBB if `sigma_m > q75`.

9. **Plot the 2D evaluation and quadrant map.** Create a scatter plot with mu on the x-axis and sigma on the y-axis. Draw median lines to define quadrants. Label each point with model name and credit grade. Q1 (high-mu, low-sigma) is the recommended deployment zone.

10. **Make regime-specific selection recommendations.** For agentic or multi-step pipelines: prioritize low sigma first, then select for mu within acceptable grade tiers (AAA/AA preferred). For single-shot controlled environments: mu can weigh more heavily, but Q4 models still require caution. Report the grade alongside mu to give stakeholders a risk-aware selection language.

## Concrete Examples

**Example 1: Selecting a model for an agentic coding assistant**

User: "I'm building a multi-step coding agent that uses tool calls. I've narrowed it down to three models that score similarly on HumanEval. Which should I deploy?"

Approach:
1. Design 6 system prompt templates covering realistic agent variations:
   - "You are a coding assistant. Output function code only, no explanation."
   - "You are a senior developer. Think through the problem, then write clean code."
   - "Respond with code inside ```python``` blocks. Include brief comments."
   - "Output a JSON object with fields: 'reasoning', 'code', 'tests'."
   - "Be concise. Write the function and nothing else."
   - "First verify the requirements, then implement step by step."
2. Run all three models across all 6 templates on a fixed 300-item coding benchmark subset.
3. Compute mu and sigma for each model.

Output:
```
Model Evaluation Report (CreditAudit)
======================================
Benchmark: HumanEval (300-item fixed subset, 6 templates)

Model           | mu (%) | sigma (%) | Grade | Quadrant
----------------|--------|-----------|-------|----------
Model-A         | 72.3   | 1.1       | AAA   | Q1 (Safe default)
Model-B         | 73.8   | 3.4       | BBB   | Q4 (Strong but fragile)
Model-C         | 68.5   | 1.5       | AA    | Q2 (Predictable baseline)

Neutrality check: S_bar_t range = [70.1, 72.0] -- PASS (no template drift)

Recommendation: For agentic pipelines, deploy Model-A (AAA).
Model-B scores 1.5pp higher on average but has 3x the fluctuation.
In a 10-step agent chain, Model-B's instability compounds --
a single format violation breaks downstream parsing.
Model-A's stability reduces integration cost and monitoring overhead.
```

**Example 2: Building a CreditAudit evaluation harness in Python**

User: "Help me write a script that runs CreditAudit on my models."

Approach:
1. Structure the evaluation as a nested loop over models, templates, and benchmark items.
2. Compute per-model statistics and grades.

Output:
```python
import numpy as np
from typing import Dict, List

def creditaudit(
    models: List[str],
    templates: List[str],
    benchmark_items: Dict[str, List[dict]],  # {bench_name: [{question, answer}]}
    run_model_fn,  # (model, template, question) -> response
    parse_fn,      # (response) -> extracted_answer
) -> Dict[str, dict]:
    """Run CreditAudit evaluation and return per-model results."""
    results = {}
    all_sigmas = []

    for model in models:
        overall_scores = []  # one S_m,t per template
        bench_details = {}

        for t_idx, template in enumerate(templates):
            bench_scores = []
            for bench_name, items in benchmark_items.items():
                correct = 0
                for item in items:
                    response = run_model_fn(model, template, item["question"])
                    predicted = parse_fn(response)
                    if predicted == item["answer"]:
                        correct += 1
                s_mtb = correct / len(items)
                bench_scores.append(s_mtb)
                bench_details.setdefault(bench_name, []).append(s_mtb)

            s_mt = np.mean(bench_scores)  # overall score for this template
            overall_scores.append(s_mt)

        mu = np.mean(overall_scores)
        sigma = np.std(overall_scores, ddof=1)  # sample std
        all_sigmas.append(sigma)
        results[model] = {"mu": mu, "sigma": sigma, "scores": overall_scores, "benchmarks": bench_details}

    # Assign grades via cross-model quantiles
    q25, q50, q75 = np.quantile(all_sigmas, [0.25, 0.50, 0.75])
    for model in models:
        s = results[model]["sigma"]
        if s <= q25:
            results[model]["grade"] = "AAA"
        elif s <= q50:
            results[model]["grade"] = "AA"
        elif s <= q75:
            results[model]["grade"] = "A"
        else:
            results[model]["grade"] = "BBB"

    # Neutrality diagnostic
    n_templates = len(templates)
    s_bar = []
    for t_idx in range(n_templates):
        t_scores = [results[m]["scores"][t_idx] for m in models]
        s_bar.append(np.mean(t_scores))
    drift_range = max(s_bar) - min(s_bar)
    neutrality_pass = drift_range < 0.05  # 5pp threshold

    return {"models": results, "quantiles": {"q25": q25, "q50": q50, "q75": q75},
            "neutrality": {"s_bar": s_bar, "drift_range": drift_range, "pass": neutrality_pass}}
```

**Example 3: Interpreting a CreditAudit report for stakeholders**

User: "Our team ran CreditAudit and Model X got AA with mu=71% while Model Y got BBB with mu=74%. Leadership wants to go with Model Y because of the higher score. How do I push back?"

Approach:
1. Explain the 2D framework and what BBB means in operational terms.
2. Quantify the deployment cost difference.

Output:
```
CreditAudit Risk Assessment: Model X (AA) vs Model Y (BBB)
============================================================

Model Y scores 3pp higher on average but sits in Quadrant Q4
(strong but scenario-fragile). Here's what that means operationally:

1. STABILITY GAP: Model Y's BBB grade means its sigma is in the top
   25% of all evaluated models. Under routine prompt changes (format
   tweaks, tone adjustments, output schema updates), Model Y's accuracy
   swings significantly more than Model X.

2. AGENTIC RISK: In multi-step pipelines, instability compounds.
   If Model Y has a 5% chance of format violation per step, a 10-step
   chain sees ~40% pipeline failure rate. Model X's lower sigma means
   tighter per-step reliability.

3. OPERATIONAL COST: BBB models require heavier prompt hardening,
   more regression testing on prompt changes, larger scenario coverage,
   more conservative rollouts, and more human fallback capacity.

4. RECOMMENDATION: Deploy Model X (AA) for production pipelines.
   Reserve Model Y for controlled, single-shot use cases where its
   higher peak score matters and prompt variation is locked down.

The 3pp accuracy difference is dwarfed by the integration and
maintenance cost of Model Y's instability in production.
```

## Best Practices

- **Do:** Use at least 5 semantically diverse system prompt templates. Fewer than 5 produces unreliable sigma estimates. Templates should cover realistic variations: format constraints, verbosity levels, reasoning instructions, and output schemas.
- **Do:** Fix the evaluation subset and record the sampling seed. CreditAudit isolates prompt-induced variation from data sampling noise only when every model sees the exact same items.
- **Do:** Always run the neutrality diagnostic before interpreting grades. If cross-model mean scores shift substantially across templates (>5pp range), some templates are acting as difficulty manipulators rather than protocol variants.
- **Do:** Report both mu and grade together, never grade alone. A BBB model with mu=85% is a very different proposition from a BBB model with mu=55%.
- **Avoid:** Using adversarial or jailbreak prompts as templates. CreditAudit measures sensitivity to *routine* deployment variations, not adversarial robustness. Adversarial prompts produce artificially inflated sigma.
- **Avoid:** Grading with fewer than 4 models. Cross-model quantiles need a reasonable pool to produce meaningful grade boundaries. With 2-3 models the quartile cuts are unstable.

## Error Handling

- **Template difficulty drift detected (neutrality check fails):** Remove or rewrite templates where `S_bar_t` deviates significantly from the others. A template that is systematically harder (not just harder for some models) contaminates sigma with irrelevant difficulty signal. Re-run the audit after revision.
- **Sigma values are all near zero:** Templates are too similar to each other. Increase semantic diversity -- add templates that change output format, reasoning style, or verbosity requirements.
- **Parser failures inflate/deflate scores:** If the deterministic parser `g()` cannot extract an answer from a response, decide on a consistent policy (count as incorrect, or exclude). Document the policy and apply it uniformly. Parser failures that cluster on specific templates may indicate those templates induce poorly formatted output -- this is itself a valid instability signal.
- **Too few benchmark items produce noisy S_m,t,b:** Use at least 200 items per benchmark. Smaller subsets introduce sampling variance that gets conflated with prompt-induced fluctuation.

## Limitations

- CreditAudit measures sensitivity to system prompt variation only. It does not capture instability from temperature sampling, few-shot example changes, or user message phrasing variation.
- The credit grades are relative (quantile-based), not absolute. Adding or removing models from the evaluation pool changes grade boundaries. A model graded AAA in a pool of 5 might become AA in a pool of 20.
- The framework assumes accuracy as the scoring function. For generative tasks without clear ground truth (summarization, creative writing), you need a proxy scorer, which introduces its own variance.
- Template design is manual and subjective. Different template families may produce different sigma rankings. The paper recommends semantic alignment but does not provide an automated template generation method.
- Equal weighting across benchmarks in the overall score may not match deployment priorities. Weight benchmarks according to your use case if one capability matters more than others.

## Reference

**Paper:** [CreditAudit: 2nd Dimension for LLM Evaluation and Selection](https://arxiv.org/abs/2602.02515v2) (Song et al., 2026). Focus on Section 3 (framework formulation), Table 1 (grade assignments with observed quantiles q25=1.30, q50=1.57, q75=2.04), Section 5.4 (quadrant-based deployment guidance), and Appendix A.1 (neutrality diagnostic).

**Code:** [github.com/LLwork8888/CreditAudit](https://github.com/LLwork8888/CreditAudit) -- reference implementation with evaluation runner, Gradio frontend, and reporting tools.