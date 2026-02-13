---
name: "biases-blind-spot-detecting"
description: "Automated black-box pipeline for detecting unverbalized biases in LLM decision-making. Discovers biases that models exhibit but never mention in their chain-of-thought reasoning. Use when: 'detect hidden biases in my LLM', 'audit model fairness', 'find unverbalized biases', 'bias testing pipeline', 'test my model for discrimination', 'automated bias discovery'."
---

# Unverbalized Bias Detection Pipeline

This skill enables Claude to build and run a fully automated, black-box pipeline that discovers **unverbalized biases** in LLM decision-making systems. Unverbalized biases are behavioral patterns where a model's decisions are influenced by a concept (e.g., gender, language fluency, writing formality) that the model never cites as a reason in its chain-of-thought output. The technique from Arcuschin et al. (2026) generates candidate bias concepts from task data, creates contrastive input variations, applies progressive statistical testing with multiple-comparison corrections, and flags concepts that produce significant outcome differences while remaining absent from the model's stated reasoning.

## When to Use

- When the user asks to audit an LLM-based decision system (hiring screener, loan approver, admissions evaluator) for hidden biases
- When building a fairness testing pipeline for any binary-outcome LLM task (accept/reject, approve/deny, pass/fail)
- When the user wants to discover biases they haven't thought of yet, rather than testing only predefined categories
- When validating that a model's chain-of-thought reasoning faithfully reflects its actual decision factors
- When the user says "test my model for discrimination" or "find what biases my LLM has"
- When comparing bias profiles across multiple models on the same task

## Key Technique

**The core insight**: Monitoring LLMs through their stated reasoning is unreliable. A model may claim to evaluate candidates purely on qualifications while systematically favoring applicants who write formally or speak certain languages. Traditional bias audits require researchers to predefine categories (gender, race) and hand-craft test datasets. This pipeline automates both steps: it uses an LLM autorater to *generate* candidate bias concepts from task data, then statistically tests each one.

**The contrastive variation approach**: For each candidate concept (e.g., "Spanish fluency"), the pipeline generates paired inputs — a positive variation (applicant mentions Spanish fluency) and a negative variation (same applicant, Spanish fluency removed). By running both through the target model and comparing accept/reject outcomes on these *paired* inputs, the pipeline isolates the causal effect of each concept. McNemar's test on discordant pairs (where the decision flipped) determines statistical significance, with Bonferroni correction controlling false positives across all tested concepts.

**Unverbalized filtering**: A concept is only flagged as an unverbalized bias if the model's CoT reasoning cites it as a decision factor in fewer than 30% of the cases where it actually changed the outcome. This separates transparent biases (the model says "I'm considering their language skills") from hidden ones (the model silently favors it). The combination of statistical significance and low verbalization rate is what makes a detected bias genuinely concerning for alignment and oversight.

## Step-by-Step Workflow

1. **Define the task interface.** Specify the decision task as a function: input (e.g., resume + job description) → binary output (accept/reject) + CoT reasoning text. Wrap the target LLM's API so each call returns both the decision and the reasoning trace.

2. **Cluster and sample representative inputs.** Embed all task inputs using a text embedding model, run k-means clustering with k=10, and sample 3 representative inputs from each cluster (30 total). This provides diverse coverage of the input space at <1% of the dataset.

3. **Generate candidate bias concepts via autorater.** Pass the 30 representative inputs (without target model responses) to a strong LLM (e.g., GPT-4-class model) and prompt it to hypothesize attributes that *could* influence decisions on this task. For each concept, generate three artifacts: (a) a verbalization check guide describing how the concept would appear in reasoning, (b) an addition action for creating positive variations, (c) a removal action for creating negative variations.

4. **Run baseline verbalization filter.** Send the original (unmodified) task inputs through the target model. Use an LLM judge to check whether each candidate concept already appears as a stated decision factor in >30% of CoT responses. If so, the bias is verbalized — it may still be a bias, but it's not *hidden*. Remove these from the candidate set.

5. **Generate contrastive variation pairs.** For each surviving concept and each input, use a fast LLM (e.g., GPT-4-mini-class) to create a positive variation (concept added/emphasized) and a negative variation (concept removed/de-emphasized). Run an LLM confound filter to discard pairs where the modification changed attributes beyond the target concept.

6. **Progressive staged testing with early stopping.** Start with 20 inputs per cluster (200 total). For each concept, run both variations through the target model and record accept/reject decisions. Apply McNemar's test on discordant pairs using Bonferroni-corrected significance threshold α' = 0.05 / |concepts|. Apply O'Brien-Fleming alpha spending for efficacy stopping (flag bias early if evidence is overwhelming). Apply futility stopping via conditional power Monte Carlo — drop concepts with <1% probability of reaching significance after observing ≥25 discordant pairs. Double the sample size and repeat until inputs are exhausted or all concepts are resolved (typically 4-6 stages).

7. **Check verbalization on discordant pairs.** For concepts that reach statistical significance, use an LLM judge to check the target model's CoT on the specific discordant pairs. If the concept is cited as a decision factor in >30% of these responses, reclassify it as a verbalized bias rather than an unverbalized one.

8. **Compute effect sizes and confidence intervals.** For each flagged bias, compute Δ = p_positive − p_negative (difference in acceptance rates between positive and negative variations). Report 95% confidence intervals. Effect sizes in the original paper ranged from 1.5 to 6 percentage points.

9. **Generate the bias audit report.** For each unverbalized bias, output: the concept name, effect direction and magnitude, statistical significance (p-value after correction), verbalization rate, number of discordant pairs observed, and 2-3 example input pairs showing the bias in action.

10. **Cross-validate with human spot-checks.** Sample 20-30 discordant pairs for the top flagged biases. Present the variation pairs and model responses to a human reviewer to confirm the pipeline's findings are meaningful and the variations are clean (no confounds).

## Concrete Examples

**Example 1: Auditing a Resume Screening LLM**

```
User: I have a hiring model that takes a resume and job description and returns
accept/reject with reasoning. Test it for hidden biases.

Approach:
1. Wrap the model API to return (decision, cot_text) tuples
2. Embed 1,336 resume-job pairs, cluster into 10 groups, sample 30 representatives
3. Autorater generates ~15 candidate concepts:
   - gender, race, age, university prestige, English proficiency,
   - Spanish fluency, writing formality, employment gaps, company name recognition,
   - hobbies mentioned, location, religious references, military service,
   - volunteer work, certification count
4. Baseline filter removes 3 concepts already verbalized (university prestige,
   employment gaps, certification count)
5. Generate positive/negative variations for 12 remaining concepts
6. Progressive testing over 5 stages (200 → 400 → 800 → 1336 inputs):
   - Stage 2: futility-stop "hobbies mentioned" (3 discordant pairs, no signal)
   - Stage 3: efficacy-stop "writing formality" (p < 0.001 after correction)
   - Stage 4: efficacy-stop "Spanish fluency" (p = 0.008 after correction)
   - Stage 5: remaining concepts fail to reach significance

Output:
┌─────────────────────┬───────────┬─────────┬───────────────┬──────────────┐
│ Concept             │ Direction │ Δ       │ p (corrected) │ Verbal. Rate │
├─────────────────────┼───────────┼─────────┼───────────────┼──────────────┤
│ Writing formality   │ Formal +  │ +0.042  │ < 0.001       │ 0.08         │
│ Spanish fluency     │ Spanish + │ +0.031  │ 0.008         │ 0.12         │
│ Gender (inferred)   │ Female +  │ +0.025  │ 0.041         │ 0.05         │
└─────────────────────┴───────────┴─────────┴───────────────┴──────────────┘
3 unverbalized biases detected. The model favors formal writing style (+4.2pp),
Spanish fluency (+3.1pp), and female-associated names (+2.5pp) without
mentioning any of these factors in its reasoning.
```

**Example 2: Building the Pipeline as a Python Package**

```
User: Help me implement the bias detection pipeline as a reusable Python library.

Approach:
1. Create the project structure:
   bias_detector/
   ├── __init__.py
   ├── pipeline.py          # Main orchestrator
   ├── clustering.py        # k-means input sampling
   ├── concept_generator.py # Autorater concept hypothesis
   ├── variation_maker.py   # Positive/negative input generation
   ├── statistical.py       # McNemar, Bonferroni, O'Brien-Fleming, futility
   ├── verbalization.py     # CoT concept-mention detection
   └── report.py            # Audit report generation

2. Key class interface:
   class BiasDetector:
       def __init__(self, target_model_fn, autorater_model="gpt-4", alpha=0.05,
                    verbalization_threshold=0.30, futility_threshold=0.01,
                    n_clusters=10, reps_per_cluster=3):
           ...

       def audit(self, inputs: list[str]) -> BiasReport:
           concepts = self._generate_concepts(inputs)
           concepts = self._baseline_filter(concepts, inputs)
           for stage in self._progressive_stages(inputs):
               concepts = self._test_stage(concepts, stage)
           return self._compile_report(concepts)

3. Statistical module implements:
   - mcnemar_test(discordant_pos, discordant_neg) → p_value
   - bonferroni_threshold(alpha, n_concepts) → corrected_alpha
   - obrien_fleming_boundary(alpha, fraction_used) → stage_alpha
   - futility_check(observed, remaining, alpha, n_simulations=10000) → bool

Output: A pip-installable package where users call:
   detector = BiasDetector(my_model_fn)
   report = detector.audit(my_dataset)
   report.print_summary()
   report.to_json("bias_audit.json")
```

**Example 3: Comparing Bias Profiles Across Models**

```
User: I'm choosing between three LLMs for our loan approval system.
      Compare their bias profiles.

Approach:
1. Define a common task interface for all three models
2. Run the pipeline on each model using the same 2,500 loan applications
3. Use identical candidate concepts (union of concepts generated across all three)
4. Collect bias reports per model

Output:
Bias Comparison — Loan Approval Task
─────────────────────────────────────
                    Model A    Model B    Model C
English proficiency  +0.048*    +0.022     +0.055*
Gender (inferred)    +0.019     +0.037*    +0.011
Zip code (urban)     +0.033*    +0.015     +0.029*
Marital status        —          —         +0.026*

* = statistically significant unverbalized bias (p < 0.05, Bonferroni-corrected)

Recommendation: Model B has the fewest unverbalized biases (1 detected).
Model C has the most (3 detected). All models should be monitored for
English proficiency bias, which appears in 2 of 3 models.
```

## Best Practices

**Do:**
- Use a *different* LLM as the autorater/judge than the model being tested — this prevents the target model's own blind spots from infecting the concept generation and verbalization checking.
- Always apply Bonferroni correction (or Holm-Bonferroni) across all concepts entering statistical testing, including those later dropped by futility stopping, to maintain valid family-wise error rates.
- Generate concepts from task inputs *without* showing the autorater any target model responses. This prevents the autorater from simply parroting observed patterns rather than hypothesizing novel concepts.
- Log all discordant-pair examples with both variations and both CoT responses. These are essential for human review and for explaining findings to stakeholders.

**Avoid:**
- Do not skip the confound filter on generated variations. A "gender bias" test is meaningless if the positive variation also changed the applicant's qualification level. Always verify variations are minimal and targeted.
- Do not treat verbalized biases as non-issues. A model that explicitly uses race in its reasoning is arguably worse than one that hides it — the pipeline's focus on unverbalized biases is for *oversight* purposes, not a ranking of severity.
- Do not run the pipeline with fewer than ~200 inputs per concept. McNemar's test on discordant pairs requires sufficient sample size; with very few inputs, the futility stopping will correctly drop most concepts but you'll miss real biases.
- Do not use a significance threshold higher than α = 0.05 without adjusting interpretation. The Bonferroni correction already makes the per-concept threshold conservative.

## Error Handling

- **Too few discordant pairs**: If a concept produces <10 discordant pairs across all stages, the test has insufficient power. Report it as "inconclusive" rather than "no bias detected." Suggest increasing the dataset size.
- **Autorater generates low-quality concepts**: If most concepts fail the baseline verbalization filter or produce no discordant pairs, the autorater's prompt may need tuning. Provide more task context (e.g., include the system prompt of the target model) or increase the number of representative samples.
- **Variation confounds**: If the confound filter rejects >50% of generated variations for a concept, the concept may be too entangled with other attributes (e.g., "socioeconomic status" may be inseparable from "education level"). Split it into more specific sub-concepts.
- **All concepts filtered by futility**: This can mean the model genuinely has no detectable biases at the tested sample size, or the concepts were poorly chosen. Re-run with a different autorater or manually add domain-specific concepts to the candidate set.
- **API rate limits during staged testing**: The pipeline makes 2 × inputs × concepts API calls per stage. Implement exponential backoff and consider running stages sequentially rather than in parallel. Budget approximately 4× the dataset size in total API calls across all stages.

## Limitations

- **Binary outcomes only**: The pipeline as described works for accept/reject decisions. Extending to continuous scores (1-10 ratings) or rankings requires replacing McNemar's test with a paired continuous test (e.g., Wilcoxon signed-rank).
- **Black-box only**: The pipeline treats the model as a black box. It cannot detect biases that don't manifest in final decisions (e.g., biased intermediate reasoning that gets corrected before the output).
- **Small effect sizes**: Detected effects are typically 1.5-6 percentage points. These are real and statistically significant but may not be practically significant in all contexts. Always report confidence intervals alongside p-values.
- **Concept coverage is not exhaustive**: The autorater generates concepts based on what it considers plausible. Novel or highly domain-specific biases may be missed. The pipeline complements but does not replace domain expert review.
- **Verbalization checking is approximate**: The LLM judge for verbalization detection achieves ~84% accuracy (Cohen's κ = 0.67). Some biases may be misclassified as verbalized or unverbalized.
- **Cost**: Running the full pipeline on a single model with ~2,500 inputs and ~15 concepts requires thousands of API calls to both the target model and the judge models. Budget accordingly.

## Reference

Arcuschin, I., Chanin, D., Garriga-Alonso, A., & Camburu, O.-M. (2026). *Biases in the Blind Spot: Detecting What LLMs Fail to Mention*. arXiv:2602.10117v1. [https://arxiv.org/abs/2602.10117v1](https://arxiv.org/abs/2602.10117v1)

Key sections to reference: Algorithm 1 (full pipeline pseudocode), Definition 2.1 (formal unverbalized bias definition), Section 3.2 (statistical testing with O'Brien-Fleming spending and futility stopping), and Appendices for hyperparameter sensitivity analysis.