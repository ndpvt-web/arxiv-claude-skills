---
name: "when-better-prompts-hurt"
description: "Evaluation-driven prompt iteration using the Define-Test-Diagnose-Fix loop and Minimum Viable Evaluation Suite (MVES). Prevents regressions when changing LLM prompts by building structured test suites before iterating. Use when: 'evaluate my prompts', 'my prompt change broke something', 'build a test suite for my LLM app', 'why did my improved prompt make results worse', 'set up eval for my RAG pipeline', 'create evaluation harness for my agent'."
---

This skill enables Claude to apply evaluation-driven prompt development based on the Define-Test-Diagnose-Fix methodology. Instead of blindly swapping in "better" prompt templates, Claude builds a Minimum Viable Evaluation Suite (MVES) first, measures baseline performance across multiple quality dimensions, then iterates with evidence. The core insight: generic prompt improvements are not monotonic -- a prompt that improves instruction-following can simultaneously degrade extraction accuracy by 10% and RAG compliance by 13%. This skill teaches Claude to catch those trade-offs before they reach production.

## When to Use

- When the user wants to change or "improve" a prompt and needs to know if the change actually helps across all dimensions
- When the user reports that a prompt change broke structured output (JSON extraction, schema compliance, citation formatting)
- When the user asks to build an evaluation harness or test suite for an LLM application, RAG pipeline, or agentic workflow
- When the user is comparing two prompt variants and needs a rigorous methodology beyond "which one looks better"
- When the user has a RAG system producing "correct but unsupported" answers -- factually right but not grounded in retrieved context
- When the user needs to set up LLM-as-judge evaluation and wants to avoid known bias pitfalls (position bias, verbosity bias, self-preference)
- When the user is debugging why a "helpful assistant" system prompt degraded their structured extraction task

## Key Technique

The paper introduces the **Define-Test-Diagnose-Fix** loop as a replacement for ad-hoc prompt tweaking. The critical observation is that LLM quality is multi-dimensional -- correctness, helpfulness, harmlessness, groundedness, format adherence, refusal correctness, and consistency can trade off against each other. A prompt that sounds more "helpful" may sacrifice format strictness. A prompt that improves safety may over-refuse legitimate queries. Without a structured evaluation suite measuring each dimension, you cannot detect these trade-offs.

The **Minimum Viable Evaluation Suite (MVES)** is a tiered framework. MVES-Core (for any LLM app) requires a golden set of 50-200 version-controlled test cases, stratified across intents with ~20% adversarial inputs, plus automated assertions for format, required fields, and prohibited content. MVES-RAG adds retrieval quality metrics (Recall@k, MRR), faithfulness/groundedness checks, and explicit tests for "correct but unsupported" answers. MVES-Agentic adds multi-step trajectory evaluation, per-tool success rates, and sandboxed execution verification.

The experimental evidence is concrete: replacing task-specific extraction prompts with a generic "helpful assistant" template on Llama 3 8B dropped extraction pass rate from 100% to 90% and RAG compliance from 93.3% to 80%, while instruction-following improved from 73% to 86%. Ablation showed the degradation came specifically from generic rules conflicting with task-specific constraints -- not from the system wrapper itself. This proves that prompt changes must be validated against task-specific test suites, not assumed beneficial.

## Step-by-Step Workflow

1. **Define quality dimensions**: Identify which of the seven dimensions matter for this specific application -- correctness, helpfulness, harmlessness, groundedness, refusal correctness, format adherence, consistency. Rank them by cost-of-failure (e.g., a medical Q&A app ranks correctness above helpfulness; a JSON API ranks format adherence above verbosity).

2. **Build the golden test set**: Create 50-200 test cases, version-controlled alongside the prompt. Stratify by intent (all user request categories represented), difficulty (50% easy, 30% medium, 20% hard), and include adversarial inputs (~20% of total: ambiguous queries, out-of-scope requests, prompt injections, format-breaking inputs).

3. **Write automated assertions for each test case**: For structured output, assert JSON validity, required keys, value types. For RAG, assert citation presence, source-only grounding, no hallucinated references. For agents, assert correct tool selection, valid parameters, proper sequencing. Express each assertion as a pass/fail check.

4. **Run the baseline evaluation**: Execute all test cases against the current prompt. Record two metrics: **all-pass rate** (percentage of cases where every assertion passes) and **check-pass rate** (micro-average across all individual checks). Store results as the baseline snapshot.

5. **Make one prompt change at a time**: Modify only one variable (system prompt wording, few-shot examples, constraint language, or model). Never change multiple variables simultaneously -- you need to attribute any performance shift to a specific cause.

6. **Run the comparison evaluation**: Execute the same test suite against the modified prompt. Compare all-pass rate and check-pass rate per quality dimension. Flag any dimension that regressed by more than 5 percentage points.

7. **Diagnose regressions by failure category**: Group failures into patterns -- is the model adding conversational filler to structured output? Over-refusing? Ignoring grounding constraints? Hallucinating citations? The pattern determines the fix. Generic rules conflicting with task-specific constraints is the most common cause of regression.

8. **Fix with targeted constraints, not broader rules**: If extraction broke, add explicit format constraints ("Output VALID JSON ONLY. Do not include markdown formatting.") rather than making the overall prompt more restrictive. If RAG grounding broke, add "Only use information from the provided context. If the context does not contain the answer, say so."

9. **Re-run and confirm no new regressions**: After the fix, run the full suite again. Confirm the targeted fix resolved the regression without degrading other dimensions. Only then accept the prompt change.

10. **Add production failures to the living test suite**: When real users hit failures, add those cases to the golden set. This prevents the same failure class from recurring and grows coverage over time.

## Concrete Examples

**Example 1: Evaluating a prompt change for a JSON extraction API**

User: "I want to improve my extraction prompt. Currently it says 'Extract the following fields as JSON: name, email, phone.' I found a better template that adds 'You are a helpful, accurate assistant. Think step by step before answering. Follow best practices for data extraction.'"

Approach:
1. Build a 20-case golden set: 10 clean inputs, 5 with missing fields, 3 with ambiguous formatting, 2 adversarial (embedded instructions, special characters)
2. Define assertions: JSON parses successfully, all required keys present, no markdown wrappers, no conversational preamble, values match expected output
3. Run baseline with original prompt -- record: all-pass rate, JSON validity rate, key completeness rate
4. Run comparison with "improved" prompt -- record same metrics
5. Compare dimension by dimension

Output:
```
## Evaluation Report: Extraction Prompt Change

Baseline (task-specific prompt):
  JSON validity:     20/20 (100%)
  Key completeness:  20/20 (100%)
  Value accuracy:    18/20 (90%)
  All-pass rate:     18/20 (90%)

Candidate (generic "helpful assistant" prompt):
  JSON validity:     16/20 (80%)  [REGRESSION: -20%]
  Key completeness:  18/20 (90%)  [REGRESSION: -10%]
  Value accuracy:    17/20 (85%)  [REGRESSION: -5%]
  All-pass rate:     14/20 (70%)  [REGRESSION: -20%]

Failure pattern: 4 cases wrapped JSON in ```json markdown blocks.
2 cases included "Here is the extracted data:" preamble.

Diagnosis: Generic "helpful assistant" framing encourages
conversational output style, conflicting with strict JSON requirement.

Recommendation: REJECT the prompt change. If instruction-following
improvements are desired, add them as separate constraints without
removing the "Output VALID JSON ONLY" directive.
```

**Example 2: Setting up MVES for a RAG customer support bot**

User: "I need to evaluate my RAG pipeline that answers warranty questions from our policy documents."

Approach:
1. Apply MVES-RAG tier: start with MVES-Core, add RAG-specific checks
2. Build stratified test set from warranty domain
3. Include the critical "correct but unsupported" test category

Output:
```
## MVES-RAG Test Suite: Warranty Support Bot

### Test Set Structure (60 cases)
- 20 direct-answer questions (answer clearly in policy docs)
- 10 multi-document questions (require combining 2+ sections)
- 10 unanswerable questions (answer NOT in provided docs)
- 10 "correct but unsupported" traps (common knowledge answers
  that happen to be right but aren't in the retrieved context)
- 10 adversarial (ambiguous warranty terms, edge cases,
  prompt injections like "ignore your instructions")

### Assertion Checklist Per Case
Core:
  [ ] Response is well-formed text (no raw errors)
  [ ] Response addresses the user's question
  [ ] No prohibited content (competitor mentions, legal advice)
RAG-specific:
  [ ] Citations present and reference only provided sources
  [ ] No fabricated document names or section numbers
  [ ] Unanswerable questions get "I don't have that information"
  [ ] "Correct but unsupported" cases cite sources, not parametric knowledge
  [ ] Retrieved context contains relevant passages (Recall@3 >= 0.8)

### Metrics to Track
  - Faithfulness: fraction of claims traceable to retrieved context
  - Citation compliance: percentage with valid source references
  - Refusal accuracy: correct refusal on unanswerable questions
  - All-pass rate across all assertion categories
```

**Example 3: Diagnosing why an agent workflow degraded after a prompt update**

User: "I updated my agent's system prompt to be more detailed about tool usage, but now it's calling the wrong tools 15% of the time."

Approach:
1. Apply MVES-Agentic tier
2. Build trajectory-level test cases capturing multi-step tool sequences
3. Run before/after comparison on per-tool success rates

Output:
```
## Diagnosis: Agent Tool Selection Regression

Before (concise prompt):
  Correct tool selection:  95% (19/20 trajectories)
  Correct parameters:      90% (18/20)
  Correct sequencing:      85% (17/20)

After (detailed prompt):
  Correct tool selection:  80% (16/20) [REGRESSION: -15%]
  Correct parameters:      95% (19/20) [IMPROVED: +5%]
  Correct sequencing:      90% (18/20) [IMPROVED: +5%]

Failure pattern: In 4 cases, the model selected a similar-but-wrong
tool. All 4 involved the detailed prompt's new section describing
tool capabilities -- the model pattern-matched on described features
rather than the actual tool name/purpose.

Root cause: The verbose tool descriptions introduced ambiguity
between tools with overlapping capability descriptions.

Fix: Keep the detailed parameter instructions (which improved
accuracy) but revert to concise, distinctive tool descriptions.
Re-run evaluation to confirm.
```

## Best Practices

- **Do:** Version-control your test suite alongside your prompts. Every prompt change gets a corresponding evaluation run stored as a snapshot.
- **Do:** Include "correct but unsupported" test cases in any RAG evaluation. These catch the most dangerous failure mode -- answers that look right but aren't grounded in retrieved context.
- **Do:** Measure multiple quality dimensions independently. A single aggregate score hides trade-offs between correctness, format adherence, and helpfulness.
- **Do:** When using LLM-as-judge, randomize response order to mitigate position bias, and calibrate against at least 25 human-labeled examples.
- **Avoid:** Assuming any prompt change is universally beneficial. The paper proves generic improvements trade off behaviors -- always validate against your specific task's test suite.
- **Avoid:** Changing multiple prompt variables at once. You lose the ability to diagnose which change caused a regression. One change per evaluation cycle.
- **Avoid:** Using only exact-match metrics for open-ended outputs, or only semantic similarity for structured outputs. Match the metric type to the output type.
- **Avoid:** Treating LLM-as-judge scores as ground truth. Known failure modes include verbosity bias (rewarding longer responses regardless of quality) and self-preference bias (favoring outputs from the same model family).

## Error Handling

**Small test suite shows no difference**: With fewer than ~400 cases, you may lack statistical power to detect a 5% regression. Report confidence intervals alongside point estimates. For preliminary development, a 50-case suite catches large regressions (>15%); expand to 200+ for production readiness.

**Conflicting dimension results**: When one dimension improves and another regresses, do not average them. Present each dimension separately to the user and ask which dimension has higher cost-of-failure for their application. The user decides the trade-off.

**LLM-as-judge disagrees with automated checks**: Trust automated checks for objective properties (JSON validity, key presence, citation format). Use LLM-as-judge only for subjective dimensions (helpfulness, tone, coherence). When they conflict, the automated check wins for structured requirements.

**Test suite becomes stale**: If production failure patterns shift but the test suite stays static, evaluation gives false confidence. Add every new production failure class to the golden set within one development cycle.

## Limitations

- This approach requires upfront investment in test case design. For one-off experimental prompts with no production deployment, the full MVES may be overkill -- a 10-case smoke test suffices.
- The paper's experiments used deterministic decoding (temperature=0). Production systems with stochastic sampling need multiple runs per test case to account for output variance, increasing evaluation cost.
- MVES does not solve the cold-start problem: if you have no real user queries yet, you must synthesize test cases, which may not reflect actual production distribution.
- LLM-as-judge correlation with human judgment (0.70-0.85) means ~15-30% disagreement. For high-stakes applications, there is no substitute for human evaluation on a calibration subset.
- The framework does not address A/B testing in production or online evaluation -- it is purely an offline development methodology.

## Reference

[When "Better" Prompts Hurt: Evaluation-Driven Iteration for LLM Applications](https://arxiv.org/abs/2601.22025v1) -- Commey, 2026. Focus on Section 4 (MVES framework tiers), Table 2 (experimental results showing prompt trade-offs), and Section 5 (judge failure modes and mitigation strategies).