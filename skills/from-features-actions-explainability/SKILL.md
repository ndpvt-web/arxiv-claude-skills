---
name: "from-features-actions-explainability"
description: "Diagnose and explain failures in agentic AI systems using trace-based rubric evaluation, bridging static feature attribution (SHAP/LIME) with trajectory-level diagnostics. Use when: 'debug why my agent failed', 'explain agent behavior', 'evaluate agent traces', 'add explainability to my agent pipeline', 'diagnose agentic failures', 'trace-based agent analysis'."
---

# From Features to Actions: Explainability for Static and Agentic AI

This skill enables Claude to apply the unified XAI evaluation framework from Chaduvula et al. (2026) to diagnose failures in both traditional ML models and multi-step agentic AI systems. The core insight: attribution methods (SHAP, LIME) work well for static predictions (Spearman rho = 0.86) but fail to diagnose execution-level failures in agentic trajectories. Instead, trace-grounded rubric evaluation — scoring agent runs against six behavioral dimensions — reliably localizes breakdowns, revealing that state tracking inconsistency is 2.7x more prevalent in failed runs and cuts success probability by 49%.

## When to Use

- When the user wants to understand why an AI agent (LLM-based tool-calling agent, ReAct agent, function-calling pipeline) failed a task
- When building evaluation harnesses for agentic systems and needing structured diagnostic rubrics
- When comparing attribution-based explainability (SHAP/LIME) against trace-level analysis to choose the right approach
- When the user asks to add post-hoc explainability to an agent pipeline that produces execution logs or tool-call traces
- When diagnosing whether failures stem from state drift, wrong tool selection, or plan incoherence in multi-step agents
- When the user wants to build a failure-mode prevalence analysis across a batch of agent runs

## Key Technique

**The problem with applying SHAP/LIME to agents:** Traditional attribution methods explain which input features drove a single prediction. Agentic systems produce trajectories — sequences of (state, action, observation) tuples — where failure emerges from compounding decisions, not a single input-output mapping. Applying SHAP to raw agent outputs yields correlational rankings but cannot localize *which step* caused failure or *why*.

**Trace-grounded rubric evaluation:** Instead of attributing outcomes to inputs, this approach evaluates each agent run against six binary behavioral dimensions derived from the execution trace alone (no outcome leakage). An LLM judge reads the full trace — tool calls, parameters, returns, intermediate states — and scores each dimension as satisfied (0) or violated (1). This produces a structured diagnostic vector per run that directly localizes failure modes.

**The bridge methodology:** To rigorously compare paradigms, the framework encodes rubric scores as binary features, trains a logistic regression predicting task success, then applies SHAP to the rubric-level features. This recovers sensible importance rankings (intent alignment: 0.473, state tracking: 0.422) but the authors emphasize these remain correlational — the rubric evaluation itself is the actionable diagnostic, not the SHAP values computed over it.

## The Six Rubric Dimensions

Each dimension receives a binary violation flag per agent run:

| Dimension | What It Tests | Failure Signal |
|---|---|---|
| **Intent Alignment** | Actions align with stated goals and task requirements | Agent pursues wrong objective |
| **Plan Adherence** | Maintains coherent multi-step plans throughout execution | Agent abandons or contradicts its own plan |
| **Tool Correctness** | Invokes tools with valid parameters and correct syntax | Malformed API calls, wrong argument types |
| **Tool Choice Accuracy** | Selects the optimal tool for each sub-task | Uses search when it should use update, etc. |
| **State Tracking Consistency** | Maintains coherent internal state across steps | Latent drift between agent memory and environment |
| **Error Recovery** | Detects and recovers from execution failures | Ignores error responses, repeats failed actions |

## Step-by-Step Workflow

### For Diagnosing Agentic Failures

1. **Collect execution traces.** Ensure the agent pipeline logs complete trajectories: each step's action (tool name + parameters), observation (tool return value), and any intermediate reasoning or state. Store as structured JSON with fields `step`, `action`, `parameters`, `observation`, `reasoning`.

2. **Define the rubric prompt template.** For each of the six dimensions, write a judge prompt that receives ONLY the trace (not the final outcome) and outputs a binary violation label with a one-sentence justification. Example for State Tracking:
   ```
   Given the following agent execution trace, determine whether the agent
   maintained consistent internal state across all steps. A violation occurs
   when the agent acts on outdated, contradictory, or fabricated state
   information. Output: {"violated": true/false, "evidence": "..."}
   ```

3. **Run the LLM judge over each trace.** Apply each rubric prompt to the full trace using a capable model (GPT-4-class or above) with low temperature (0.1) in a single pass. Collect the six binary flags per run into a diagnostic vector.

4. **Compute failure-mode prevalence.** For a batch of N runs with known outcomes, calculate P(violation | failure) and P(violation | success) for each dimension. The prevalence ratio reveals which failure modes are disproportionately associated with task failure.

5. **Compute reliability correlates.** Calculate P(success | violation) vs P(success | no violation) for each dimension. The relative risk RR = P(success|flag) / P(success|no flag) quantifies how much each violation reduces success probability.

6. **Identify the dominant failure pattern.** Classify the failure regime:
   - **"Slow failure" (state drift):** High state tracking violations, errors compound silently across steps, final step is irrecoverable. Common in structured-data tasks (databases, APIs with state).
   - **"Fast failure" (wrong tool/plan):** Single decisive error in tool choice or plan collapses the run. Common in web navigation or tasks with limited step budgets.

7. **Generate the Minimal Explanation Packet (MEP).** Bundle three components:
   - *Artifact:* The rubric violation vector + evidence sentences
   - *Linked Evidence:* The raw execution trace, tool logs, and state snapshots
   - *Verification Signals:* Prevalence ratios, reliability correlates, and (optionally) replay consistency checks

8. **Act on diagnostics.** Use the localized failure to fix the agent: add state validation checkpoints for state drift, improve tool selection prompting for tool choice failures, add plan-checking loops for plan adherence issues.

### For Static ML Explainability (Comparison Baseline)

1. Train the model and generate predictions on a held-out set.
2. Apply SHAP (KernelExplainer or LinearExplainer) and LIME to the same instances.
3. Extract top-k feature rankings from each method.
4. Compute pairwise Spearman rank correlation across methods to assess stability.
5. If rho > 0.80, attributions are stable and trustworthy for that model class.

## Concrete Examples

**Example 1: Diagnosing a failing customer-service agent**

User: "My airline booking agent succeeds only 56% of the time on TAU-bench tasks. Help me figure out why it fails."

Approach:
1. Collect the 50 execution traces (JSON logs of each tool call and response)
2. Apply the six-dimension rubric judge to each trace
3. Compute prevalence: find that state tracking violations appear in 63.6% of failures vs 23.8% of successes (ratio = 2.7x)
4. Compute reliability: success drops from 73.5% to 37.5% when state tracking is violated (RR = 0.51)
5. Examine violating traces to find the pattern: agent fetches booking details, modifies them, but subsequent actions reference the pre-modification state

Output:
```
Failure Diagnosis Report
========================
Dominant failure mode: State Tracking Inconsistency
  Prevalence ratio: 2.7x (63.6% in failures vs 23.8% in successes)
  Success impact: -36 percentage points when violated (RR = 0.51)

Pattern: Agent retrieves customer record, applies modification via
update_booking tool, but subsequent tool calls reference stale pre-update
fields (e.g., old seat assignment after seat change).

Recommendation: Add a state re-fetch step after every mutating tool call.
Validate that the agent's working memory matches the latest API response
before proceeding to the next action.
```

**Example 2: Building a rubric evaluator for a code-generation agent**

User: "I have a ReAct agent that writes and executes code. I want to add trace-based diagnostics to my eval pipeline."

Approach:
1. Adapt the six rubric dimensions to the code-generation domain:
   - Intent Alignment -> Does generated code address the stated problem?
   - Plan Adherence -> Does the agent follow its stated implementation plan?
   - Tool Correctness -> Are shell commands and file operations syntactically valid?
   - Tool Choice -> Does the agent use the right tool (write vs edit vs bash)?
   - State Tracking -> Does the agent track file contents and test results accurately?
   - Error Recovery -> Does the agent respond to test failures or syntax errors?
2. Write judge prompts for each dimension
3. Integrate into the eval loop: after each agent run, invoke the judge on the trace
4. Aggregate results across runs to find systematic weaknesses

Output (evaluation harness pseudocode):
```python
RUBRIC_DIMENSIONS = [
    "intent_alignment", "plan_adherence", "tool_correctness",
    "tool_choice", "state_tracking", "error_recovery"
]

def evaluate_run(trace: list[dict], judge_model: str) -> dict:
    results = {}
    for dim in RUBRIC_DIMENSIONS:
        prompt = load_rubric_prompt(dim)
        judgment = call_llm(judge_model, prompt + format_trace(trace))
        results[dim] = parse_binary_judgment(judgment)
    return results

def compute_failure_analysis(all_results: list[dict], outcomes: list[bool]):
    for dim in RUBRIC_DIMENSIONS:
        violations_in_failures = mean(r[dim] for r, o in zip(all_results, outcomes) if not o)
        violations_in_successes = mean(r[dim] for r, o in zip(all_results, outcomes) if o)
        ratio = violations_in_failures / max(violations_in_successes, 0.01)
        print(f"{dim}: prevalence ratio = {ratio:.1f}x")
```

**Example 3: Deciding between SHAP and trace diagnostics**

User: "Should I use SHAP or trace-based evaluation to explain my system's behavior?"

Approach:
1. Determine whether the system is static (single input -> single output) or agentic (multi-step tool-calling trajectory)
2. For static: apply SHAP + LIME, check Spearman rank correlation for stability
3. For agentic: apply trace-grounded rubric evaluation — SHAP on raw trajectories will not localize step-level failures

Output:
```
Decision Matrix
===============
Static model (classifier, regressor, single-call LLM):
  -> Use SHAP/LIME. Expect stable rankings (rho > 0.80 for linear models).
  -> Cross-validate with at least 2 methods to confirm stability.

Agentic system (multi-step, tool-calling, stateful):
  -> Use trace-based rubric evaluation with LLM judge.
  -> Attribution methods on raw features will be correlational, not causal.
  -> Optionally apply SHAP to rubric-level features as a summary view,
     but rely on per-run rubric diagnostics for actionable debugging.

Hybrid (agent with a static component, e.g., classifier gating tool calls):
  -> Use SHAP/LIME on the static component.
  -> Use trace diagnostics on the overall trajectory.
  -> Combine into a layered MEP (Minimal Explanation Packet).
```

## Best Practices

- **Do:** Evaluate traces without outcome leakage — the judge must not know whether the run succeeded or failed when scoring rubric dimensions. This prevents circular reasoning.
- **Do:** Use binary violation flags rather than continuous scores. Binary flags are more reliable for LLM judges and produce cleaner prevalence statistics.
- **Do:** Compute both prevalence ratios AND reliability correlates. A violation can be common in failures (high prevalence) but not actually predictive (low reliability impact), or vice versa.
- **Do:** Classify the failure regime (slow drift vs fast collapse) before choosing a fix. State checkpoints help slow failures; better prompting or tool routing helps fast failures.
- **Avoid:** Applying SHAP directly to raw agent action sequences or token-level features. The resulting attributions will be unstable and uninterpretable for multi-step behavior.
- **Avoid:** Using fewer than 30 runs for prevalence analysis. Small samples produce noisy ratios — the original paper uses N=50 as a minimum.

## Error Handling

- **Judge disagreement or ambiguous traces:** If the LLM judge cannot confidently assign a binary label, flag the run as "ambiguous" and exclude from prevalence statistics. Review ambiguous runs manually to refine rubric prompts.
- **All runs fail a single dimension:** If a rubric violation appears in >90% of both successes and failures, the dimension is not discriminative for this agent/task pair. Remove it from the analysis and consider whether the rubric prompt needs calibration.
- **Insufficient trace logging:** If the agent framework does not log intermediate states or tool return values, rubric evaluation will be incomplete. Ensure logging captures: action name, full parameters, full response, and any reasoning/scratchpad text.
- **Judge model limitations:** The judge model must be at least as capable as the agent model to reliably assess trace quality. If using a weaker judge, expect higher noise in rubric labels.

## Limitations

- The rubric dimensions are designed for tool-calling agents. Agents that operate through pure text generation (no tool calls) require adapted dimensions focused on reasoning coherence rather than tool correctness.
- Trace-based diagnostics are post-hoc — they explain failures after execution but do not prevent them in real time. For online intervention, the rubric would need to be applied as a runtime monitor.
- The LLM judge introduces its own biases and error rate. The paper uses GPT-5 as judge; less capable judges may produce noisier labels, particularly for subtle state tracking violations.
- Small sample sizes (N < 30) produce unreliable prevalence ratios. This approach is designed for batch evaluation, not single-run diagnosis — though individual MEPs are still informative qualitatively.
- The six dimensions may not cover all failure modes for every domain. Extend the rubric with domain-specific dimensions (e.g., "safety constraint adherence" for autonomous systems) as needed.

## Reference

**Paper:** Chaduvula, Ho, Kim, Narayanan, Alinoori. "From Features to Actions: Explainability in Traditional and Agentic AI Systems." arXiv:2602.06841v2, 2026. [https://arxiv.org/abs/2602.06841v2](https://arxiv.org/abs/2602.06841v2)

**Code:** [https://github.com/VectorInstitute/unified-xai-evaluation-framework](https://github.com/VectorInstitute/unified-xai-evaluation-framework)

**Key takeaway:** Look for Tables 7-8 (failure-mode prevalence and reliability correlates) and the MEP structure definition — these are the actionable components for building trace-based diagnostics into your own agent evaluation pipeline.