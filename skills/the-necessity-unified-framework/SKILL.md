---
name: "the-necessity-unified-framework"
description: "Design and implement standardized, reproducible evaluation harnesses for LLM-based agents. Eliminates confounding factors (system prompts, tool configs, environment drift) so benchmark results reflect true model capability. Use when: 'build an agent evaluation framework', 'make my agent benchmarks reproducible', 'standardize agent testing', 'evaluate LLM agents fairly', 'set up a sandbox for agent eval', 'create reproducible agent benchmarks'."
---

# Unified Framework for LLM Agent Evaluation

This skill enables Claude to design, build, and audit **standardized evaluation harnesses** for LLM-based agents following the framework proposed by Zhu et al. (2026). The core insight is that most agent benchmark results are confounded by extraneous factors -- system prompt differences, tool invocation formats, memory strategies, inference engine variability, and live environment drift -- rather than reflecting genuine model capability. This skill teaches how to isolate those factors by constructing a hermetically deterministic sandbox with a standardized dataset (Instruction Set, Tool Set, Environment Set), a unified agent architecture, and a multidimensional evaluation methodology covering outcome-level, process-level, and efficiency metrics.

## When to Use

- When building an evaluation harness or benchmark suite for LLM-based agents and you need results that are reproducible and comparable across models.
- When auditing an existing agent benchmark for confounding factors (prompt leakage, tool-format bias, environment non-determinism).
- When comparing two or more agent frameworks (e.g., LangChain vs. smolagents vs. custom ReAct) and you need to isolate model performance from framework effects.
- When setting up CI/CD pipelines that gate deployments on agent benchmark scores and you need deterministic pass/fail criteria.
- When designing a tool-use benchmark and you need a standardized protocol for tool definitions, parameter types, and invocation formats.
- When debugging non-reproducible agent failures caused by live web environments, API version drift, or stochastic inference behavior.

## Key Technique

**The Problem: Confounded Benchmarks.** Current agent evaluations mix together at least five independent variables: (1) inference configuration (provider safety filters, engine choice like vLLM vs. SGLang, floating-point non-associativity even at temperature=0), (2) prompting and planning strategy (lightweight vs. detailed system prompts, ReAct variants, reflection mechanisms), (3) memory formatting and management (structured vs. flat, FIFO truncation vs. summarization, long-term retrieval strategies), (4) tool invocation conventions (naming restrictions, unsupported parameter types across providers), and (5) external environment state (live URLs expiring, web content changing between runs). When a benchmark score changes, it is impossible to attribute the delta to the model vs. any of these factors.

**The Solution: Hermetic Determinism via Sandbox + Standardized Dataset + Unified Architecture.** The framework proposes three interlocking components. First, a **Standardized Dataset** comprising an Instruction Set (tasks with ground-truth criteria), a Tool Set (Python-based tool definitions using a single canonical schema), and an Environment Set (version-controlled, mocked replacements for all external dependencies). Second, a **Unified Agent Architecture** that fixes the non-model components -- inference wrapper, memory module, planner, tool executor -- so only the LLM itself varies between runs. Third, a **Multidimensional Evaluation Methodology** that goes beyond final-answer accuracy to measure outcome correctness (response + environment state diff against gold snapshot), process correctness (tool invocation trajectory vs. ground-truth path), robustness (pass@k with standardized k), efficiency (token count, latency, interaction steps), and failure taxonomy (categorized as reasoning, planning, tool-use, or environment errors).

**Why It Works.** By fixing everything except the variable under test, the framework turns agent evaluation from an observational study into a controlled experiment. The sandboxed environment eliminates the single largest source of non-reproducibility (live external state), while the unified architecture eliminates framework-specific prompt engineering as a confounder.

## Step-by-Step Workflow

1. **Audit existing confounders.** Inventory the current evaluation setup. For each of the five confounding dimensions (inference config, prompting/planning, memory, tool invocation, environment), document what varies between runs or between compared systems. Produce a confound matrix listing each factor and its current state (fixed, partially controlled, uncontrolled).

2. **Define the Instruction Set.** Write task specifications as structured objects containing: a natural-language instruction, expected input format, ground-truth answer or state-change criteria, and difficulty/category tags. Store as versioned JSON or YAML files under `eval/instructions/`.

3. **Standardize the Tool Set.** Convert all tools to a single Python-based schema with consistent naming conventions, typed parameters (avoid provider-specific unsupported types like `dict[str, Any]` on OpenAI), docstrings as descriptions, and deterministic return formats. Place tool definitions under `eval/tools/` with one file per tool.

4. **Build the Environment Set.** Replace every external dependency (web APIs, databases, file systems) with version-controlled mocks. For web-based tasks, snapshot HTML/JSON responses and serve them from a local fixture server. For database tasks, use SQLite with seeded data. Tag each environment snapshot with a semantic version. Store under `eval/environments/`.

5. **Construct the Sandbox runner.** Build a harness that: (a) loads a specific Instruction + Tool + Environment triple, (b) instantiates the agent under test using the unified architecture (fixed system prompt, fixed memory config, fixed planner), (c) injects only the LLM endpoint as the variable, (d) captures the full execution trajectory (every LLM call, tool invocation, and environment state transition), and (e) enforces determinism (pinned random seeds, temperature=0, single-threaded execution).

6. **Implement outcome-level evaluation.** Compare the agent's final response against ground-truth using exact match, fuzzy match, or a Judge-LLM (specify which and version-pin it). Also diff the final environment state against a gold-standard snapshot to catch side-effect errors.

7. **Implement process-level evaluation.** Compare the agent's tool invocation sequence against one or more ground-truth trajectories. Score using trajectory edit distance or step-wise match rate. Flag deviations as potential reasoning, planning, or tool-use errors.

8. **Implement efficiency metrics.** Record total tokens consumed (prompt + completion), wall-clock latency, and number of interaction steps (LLM calls). Report these alongside accuracy so that a model achieving 90% accuracy in 3 steps is distinguishable from one needing 15 steps.

9. **Implement robustness via pass@k.** Run each task k times (standardize k across all benchmarks, e.g., k=5). Report pass@1, pass@3, and pass@k. This captures variance from irreducible inference stochasticity.

10. **Classify failures.** For every failed task, categorize the root cause into: (a) reasoning error (wrong logical step), (b) planning error (correct reasoning but wrong action sequence), (c) tool-use error (correct plan but malformed invocation or wrong tool), or (d) environment error (correct invocation but unexpected environment response). Aggregate into a failure taxonomy report.

## Concrete Examples

**Example 1: Building a reproducible tool-use benchmark**

```
User: I have 50 tool-use tasks for evaluating agents. Results vary wildly
between runs and between frameworks. Help me make this reproducible.

Approach:
1. Audit: Identify that tasks hit live APIs (weather, search), system prompts
   differ between LangChain and smolagents, and tool schemas use different
   parameter types.
2. Environment Set: Snapshot all API responses into eval/environments/v1.0/.
   Create a local Flask fixture server that replays responses keyed by request.
3. Tool Set: Rewrite all 12 tools using a canonical Python schema:
     eval/tools/get_weather.py  -- params: (city: str, unit: str) -> dict
     eval/tools/web_search.py   -- params: (query: str, max_results: int) -> list
   Remove dict[str, Any] params; expand into explicit typed fields.
4. Instruction Set: Convert tasks to structured YAML:
     - id: "weather-001"
       instruction: "What is the weather in Tokyo in Celsius?"
       ground_truth: {"temperature": 18, "unit": "celsius"}
       tools_required: ["get_weather"]
       environment: "weather-snapshot-v1.0"
5. Sandbox: Build runner that loads triples, pins system prompt, runs agent,
   captures trajectory.
6. Evaluate: Outcome (exact match on answer), process (expected tool sequence:
   [get_weather(city="Tokyo", unit="celsius")]), efficiency (token count).

Output structure:
eval/
  instructions/           # Versioned task YAML files
  tools/                  # Canonical Python tool definitions
  environments/v1.0/      # Snapshotted API responses
  sandbox/
    runner.py             # Hermetic execution harness
    config.yaml           # Fixed system prompt, memory config, planner
  results/
    run-2026-02-10/
      outcomes.json       # Per-task pass/fail + answer
      trajectories.jsonl  # Full tool invocation logs
      efficiency.csv      # Tokens, latency, steps per task
      failures.json       # Categorized failure taxonomy
```

**Example 2: Comparing two models on an existing benchmark**

```
User: I want to compare GPT-5 vs. Claude Opus on my 200-task agent benchmark.
How do I make sure the comparison is fair?

Approach:
1. Confound matrix: Check that system prompt, tool definitions, memory strategy,
   and environment are IDENTICAL for both models. Document any provider-specific
   differences (e.g., OpenAI rejects tool names with dots; Claude allows them).
2. Normalize tool names: Replace "web.search" with "web_search" globally to
   eliminate provider-format bias.
3. Pin inference config: temperature=0, max_tokens=4096, no safety-filter
   overrides. Use the same inference wrapper for both (e.g., LiteLLM with
   identical retry/timeout settings).
4. Run pass@5 for both models on all 200 tasks.
5. Report:
   | Metric             | GPT-5      | Claude Opus |
   |--------------------|------------|-------------|
   | pass@1             | 72.5%      | 75.0%       |
   | pass@5             | 81.0%      | 83.5%       |
   | Avg tokens/task    | 2,140      | 1,870       |
   | Avg steps/task     | 4.2        | 3.8         |
   | Reasoning errors   | 18         | 14          |
   | Tool-use errors    | 9          | 7           |
   | Planning errors    | 6          | 4           |
6. Conclusion attributes performance difference to model capability because
   all confounders are controlled.
```

**Example 3: Auditing an existing benchmark for confounders**

```
User: Our team's agent benchmark gives inconsistent results. Sometimes model A
wins, sometimes model B. Help me figure out why.

Approach:
1. Run the benchmark 5 times for each model without changes. Compute variance.
   High variance (>5% accuracy swing) indicates stochastic confounders.
2. Check inference config: Are both models using the same provider endpoint?
   Finding: Model A uses vLLM, Model B uses SGLang -- different batching
   behaviors introduce divergent outputs even at temperature=0.
   Fix: Standardize on one engine or use provider API directly.
3. Check environment: Are tasks hitting live endpoints?
   Finding: 12 of 80 tasks query live web URLs. 3 URLs have gone stale.
   Fix: Snapshot all responses, serve from local fixtures.
4. Check prompts: Are system prompts identical?
   Finding: Model A gets a 200-token system prompt; Model B gets 1,400 tokens
   with chain-of-thought scaffolding baked in.
   Fix: Use one canonical system prompt for both.
5. Check tools: Are tool schemas identical?
   Finding: Model A's framework passes tool descriptions as JSON Schema;
   Model B's uses XML. Model B's format includes examples in descriptions.
   Fix: Standardize to JSON Schema, strip examples from descriptions.
6. Re-run with all fixes. Variance drops from 8% to <1%. Model B now
   consistently outperforms Model A by 3.5%, attributable to model capability.

Output: Confound audit report listing each factor, its prior state,
the fix applied, and the measured impact on result variance.
```

## Best Practices

- **Do** version-control every component of your evaluation (instructions, tools, environments, system prompts) so any result can be traced to an exact configuration snapshot.
- **Do** report efficiency metrics (tokens, steps, latency) alongside accuracy. A model that solves 80% of tasks in 3 steps is often more valuable than one solving 82% in 12 steps.
- **Do** use a standardized failure taxonomy (reasoning / planning / tool-use / environment) so error analysis is comparable across papers and teams.
- **Do** run pass@k with a fixed k and report pass@1 alongside it; a single run is insufficient given irreducible inference stochasticity.
- **Avoid** using live external services in benchmarks. Always snapshot and mock. URL expiration and content drift are the most common source of "phantom" failures.
- **Avoid** embedding chain-of-thought scaffolding, few-shot examples, or reflection mechanisms into the system prompt when comparing models -- these are confounders, not model capabilities.
- **Avoid** comparing results across different tool invocation formats (JSON Schema vs. XML vs. plain text). Normalize to one canonical format first.

## Error Handling

- **Non-deterministic results at temperature=0**: Floating-point non-associativity and batching differences cause this. Mitigate with pass@k and by pinning the inference engine version. Document the engine and hardware in results metadata.
- **Tool schema incompatibilities across providers**: Some providers reject certain parameter types (e.g., `dict[str, Any]`). Expand complex types into explicit typed fields before running evaluations.
- **Stale environment snapshots**: If tasks reference real-world data, snapshots can become semantically outdated. Tag each snapshot with a creation date and re-snapshot on a defined cadence. Never silently update snapshots mid-evaluation.
- **Judge-LLM disagreement**: When using an LLM as evaluator for open-ended responses, pin the judge model version and report inter-annotator agreement. Consider multiple judge runs and majority voting.
- **Trajectory comparison breaks on valid alternative paths**: Ground-truth trajectories may not be unique. Allow a set of acceptable trajectories or use a trajectory-similarity metric (edit distance) with a threshold rather than exact match.

## Limitations

- This framework is designed for **controlled benchmarking**, not for evaluating agents in production where environment variability is the point (e.g., real-time web browsing quality).
- Mocking external environments removes the ability to test **adaptation to novel or changing conditions**, which is itself a valuable agent capability.
- The unified architecture constraint means you cannot evaluate framework-specific innovations (novel memory architectures, custom planners) -- by design, since the goal is isolating model capability.
- Pass@k with meaningful k values (5+) multiplies compute cost linearly. Budget accordingly.
- The failure taxonomy (reasoning / planning / tool-use / environment) requires human or Judge-LLM annotation, which introduces its own subjectivity. Calibrate annotators before large-scale use.

## Reference

Zhu, P., Sun, L., Yu, P. S., & Su, S. (2026). *The Necessity of a Unified Framework for LLM-Based Agent Evaluation.* arXiv:2602.03238v1. [https://arxiv.org/abs/2602.03238v1](https://arxiv.org/abs/2602.03238v1)

Key sections to study: the confounding factor analysis (Section 3), the Sandbox architecture with Instruction/Tool/Environment Sets (Section 4), and the multidimensional evaluation methodology covering outcome-level, process-level, robustness, efficiency, and failure taxonomy metrics (Section 5).