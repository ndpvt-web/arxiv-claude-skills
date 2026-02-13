---
name: "taming-scylla-understanding-multi-headed"
description: >
  Evaluate and optimize agentic coding tool configurations using Scylla's tiered ablation
  framework and Cost-of-Pass (CoP) metric. Helps decide what level of agent complexity
  (prompts, skills, tools, multi-agent) a task actually needs, avoiding overengineering.
  Trigger phrases: "benchmark my agent setup", "evaluate agent complexity",
  "Cost-of-Pass analysis", "ablation study for agent config", "optimize agent architecture",
  "is multi-agent overkill for this task"
---

# Scylla: Tiered Ablation Framework for Agentic Coding Tools

This skill enables Claude to apply the Scylla evaluation framework from Villmow (2026) to analyze, benchmark, and right-size agentic coding configurations. The core insight is that **architectural complexity does not always improve quality** -- a maximally-configured agent with 61 skills, every tool, and hierarchical multi-agent orchestration can score *lower* and cost *3.8x more* than a selective hybrid. Scylla provides a structured method (seven tiers T0-T6, the Cost-of-Pass metric, and multi-judge consensus evaluation) to determine the minimum effective complexity for any coding task.

## When to Use This Skill

- When the user wants to decide whether a task needs a multi-agent swarm or a single focused agent
- When evaluating whether adding more skills, tools, or prompts to an agent actually improves output quality
- When the user asks "is this agent setup overkill?" or wants to reduce costs on agentic workflows
- When designing an ablation study to isolate which architectural component (prompt, skill, tool, delegation) drives results
- When benchmarking a new agentic coding tool or configuration against baselines
- When computing Cost-of-Pass to compare agent configurations on cost-efficiency, not just accuracy
- When the user notices diminishing returns or regressions from adding complexity to their agent pipeline

## Key Technique: Tiered Ablation with Cost-of-Pass

Scylla's central method is **progressive ablation across seven tiers**, each isolating a specific architectural dimension:

| Tier | Name | What It Adds | Purpose |
|------|------|-------------|---------|
| T0 | Baseline | Raw model, no/minimal system prompt | Measure innate model capability |
| T1 | Skills | Domain-specific skills (token-efficient expertise) | Measure impact of curated knowledge |
| T2 | Tooling | External tools and MCP servers via JSON schemas | Measure impact of tool access |
| T3 | Delegation | Flat multi-agent with specialist agents | Measure impact of parallelism |
| T4 | Hierarchy | Nested orchestration with self-correction loops | Measure impact of coordination depth |
| T5 | Hybrid | Selective best-of combinations from T0-T4 | Find the optimal minimal configuration |
| T6 | Super | Everything enabled (all skills, tools, agents) | Measure the cost of maximalism |

The key metric is **Cost-of-Pass (CoP)**: `CoP = total_cost / pass_rate`. This is measured in dollars and represents the expected cost to obtain one correct solution. CoP unifies accuracy and cost into a single number -- a configuration that is 95% accurate but costs $0.50 per run has CoP = $0.526, while one that is 80% accurate at $0.10 has CoP = $0.125. The cheaper, slightly-less-accurate option is 4x more cost-efficient.

The framework's critical finding is the **Token Efficiency Chasm**: T6 (everything enabled) consumed 218K cache-read tokens versus T0's 113K -- nearly double -- while scoring *lowest* among all tiers (0.943 vs 0.983). T5 (selective hybrid) achieved the frontier CoP by loading only the features that measurably contributed to quality for each specific task. This means the correct strategy is to **match tier to task difficulty**, not default to maximum configuration.

## Step-by-Step Workflow

1. **Classify the task complexity.** Determine whether the coding task is trivial (hello world, boilerplate), moderate (feature with tests, refactor), or complex (multi-file architecture, cross-system integration). This determines your starting tier.

2. **Establish a T0 baseline.** Run the task with minimal or no system prompt to measure raw model capability. Record the output quality score and token cost. This is your floor -- any added complexity must beat it.

3. **Run T1: Add domain skills only.** Inject relevant domain expertise (language idioms, framework patterns, coding standards) as concise skill documents. Measure quality and cost. Skills are token-efficient because they load targeted knowledge without schema overhead.

4. **Run T2: Add tools.** Enable external tools (linters, test runners, file search, MCP servers) via JSON schemas. Measure the delta from T1. Note: tool schemas consume tokens even when tools aren't invoked -- watch for schema bloat.

5. **Run T3: Add flat delegation.** If the task has parallelizable sub-problems, introduce specialist agents (e.g., a test-writer agent, a reviewer agent) in a flat structure. Measure whether delegation improves quality or just adds coordination overhead.

6. **Run T4 (if warranted): Add hierarchical orchestration.** Only test nested agent hierarchies with self-correction loops if T3 showed clear quality gaps. Hierarchy adds ~30% overhead versus flat delegation and rarely helps on tasks below high complexity.

7. **Compute CoP for each tier.** For each configuration: `CoP = total_cost / pass_rate`. Use at least 3-5 runs per tier to get a reliable pass rate. Plot CoP across tiers to identify the efficiency frontier.

8. **Construct T5: Selective hybrid.** Combine only the components from T0-T4 that demonstrably improved quality. Drop everything else. This is almost always the optimal configuration.

9. **Evaluate with multi-judge consensus.** Score outputs using multiple LLM judges (or a single LLM at different temperatures) across weighted rubric categories: Functional Correctness (35%), Code Quality (20%), Proportionality (15%), Overall Quality (20%), Build Pipeline (10%). Average scores across judges to reduce individual bias.

10. **Document the ablation results.** Record each tier's quality score, token usage, dollar cost, CoP, and which components were active. This creates a reusable decision matrix for future tasks of similar complexity.

## Concrete Examples

**Example 1: Deciding if a multi-agent swarm is warranted**

User: "I'm building a CLI tool that fetches weather data and formats it. Should I use a multi-agent setup with separate agents for API design, testing, and docs?"

Approach:
1. Classify: This is a moderate, single-concern task (one API, one output format).
2. T0 baseline: A raw model prompt produces functional code in one pass. Quality: 0.92. Cost: $0.08.
3. T1 with skills: Add a "CLI best practices" skill and "API error handling" skill. Quality: 0.96. Cost: $0.10.
4. T3 with delegation: Split into API agent + test agent + docs agent. Quality: 0.95. Cost: $0.31.
5. CoP comparison: T0 = $0.087, T1 = $0.104, T3 = $0.326.

Output:
```
Ablation Result:
  T0 (baseline):      Quality 0.92  Cost $0.08  CoP $0.087
  T1 (skills):        Quality 0.96  Cost $0.10  CoP $0.104
  T3 (multi-agent):   Quality 0.95  Cost $0.31  CoP $0.326

Recommendation: Use T1 (skills only). Multi-agent adds 3x cost with no
quality gain. The task is single-concern -- delegation creates coordination
overhead without parallelizable subtasks.
```

**Example 2: Evaluating a complex refactoring task**

User: "I need to migrate our Express.js API to Fastify, update all 47 route handlers, and ensure tests still pass. What's the right agent setup?"

Approach:
1. Classify: Complex, multi-file, requires cross-system consistency.
2. T0 baseline: Raw model misses edge cases in 12/47 handlers. Quality: 0.71. Cost: $0.45.
3. T1 with skills: Add "Fastify migration patterns" skill. Catches 8 more edge cases. Quality: 0.88. Cost: $0.52.
4. T2 with tools: Add test runner + linter tools. Quality jumps to 0.94. Cost: $0.68.
5. T3 with delegation: Route-handler agent + test-verification agent. Quality: 0.97. Cost: $1.20.
6. T5 hybrid: Skills (T1) + test runner tool only (from T2) + flat delegation (T3). Quality: 0.96. Cost: $0.95.

Output:
```
Ablation Result:
  T0 (baseline):      Quality 0.71  Cost $0.45  CoP $0.634
  T1 (skills):        Quality 0.88  Cost $0.52  CoP $0.591
  T2 (tools):         Quality 0.94  Cost $0.68  CoP $0.723
  T3 (delegation):    Quality 0.97  Cost $1.20  CoP $1.237
  T5 (hybrid):        Quality 0.96  Cost $0.95  CoP $0.990

Recommendation: T1 (skills) has the best CoP. If quality threshold is >0.95,
use T5 hybrid (skills + test runner + flat delegation). Skip the linter tool --
it added cost without measurable quality improvement on this task.
```

**Example 3: Quick CoP comparison for a simple task**

User: "Calculate the Cost-of-Pass for my agent config: it costs $0.15 per run and passes 82% of the time."

Approach:
1. Apply CoP formula directly.

Output:
```
CoP = $0.15 / 0.82 = $0.183

This means you pay ~$0.18 on average to get one correct solution.
To improve CoP, either reduce per-run cost or increase pass rate.
A config costing $0.20 but passing 95% would have CoP = $0.211 --
still worse. But one costing $0.12 at 78% gives CoP = $0.154 -- better.
CoP rewards cost reduction more than marginal accuracy gains.
```

## Best Practices

- **Do:** Start from T0 and add complexity incrementally. Every added component must demonstrably improve CoP, not just accuracy.
- **Do:** Use the T5 hybrid approach as your default production configuration. Cherry-pick only the skills, tools, and agents that proved their value in ablation.
- **Do:** Run at least 3-5 trials per tier to get stable pass rates. Single runs have high variance and produce unreliable CoP.
- **Do:** Weight evaluation rubrics to match task priorities. A migration task should weight Functional Correctness higher; a greenfield prototype should weight Code Quality higher.
- **Avoid:** Defaulting to T6 (everything enabled). The "Token Efficiency Chasm" shows maximalist configs consume ~2x tokens for equal or worse quality.
- **Avoid:** Adding hierarchical orchestration (T4) for tasks that don't have multi-step self-correction needs. Hierarchy adds ~30% overhead versus flat delegation with no quality gain on most tasks.
- **Avoid:** Treating accuracy as the only metric. A 98% accurate config at $2/run (CoP = $2.04) is far worse than 90% at $0.10 (CoP = $0.111) for most practical purposes.

## Error Handling

- **Pass rate of zero:** If no configuration produces a correct result, CoP is undefined (division by zero). Fall back to qualitative rubric scores and investigate whether the task exceeds the model's capability ceiling regardless of architecture.
- **High variance across runs:** If pass rate swings wildly between trials, increase trial count or investigate non-deterministic factors (API timeouts, tool flakiness, context window overflow).
- **T5 underperforms T1:** This can happen when "selective" combination introduces conflicting instructions from different tiers. Strip back to T1 and add components one at a time to find the conflict.
- **Judge disagreement:** If multi-judge scores diverge by more than 0.15, inspect the divergent rubric category. Conservative judges (like Opus) may penalize style while generous judges (like Haiku) inflate scores. Weight the median, not the mean, when disagreement is high.
- **Schema bloat in T2:** If adding tools increases cost without quality gain, check whether unused tool schemas are consuming context tokens. Prune tool definitions to only those the agent actually invokes.

## Limitations

- **Task-dependent results:** Ablation results from one task do not transfer to different task types. A configuration optimal for CLI tools may be suboptimal for web app development. Repeat ablation for each task category.
- **Model-specific:** CoP and tier rankings may differ across model families. Results demonstrated with Claude Sonnet 4.5 may not hold for GPT, Gemini, or open-weight models.
- **Cost estimation:** Token costs depend on pricing tiers, caching behavior, and prompt structure. CoP comparisons are only valid when measured under identical pricing conditions.
- **Single-vendor judges:** Using judges from the same vendor as the coding agent introduces potential scoring bias. Cross-vendor evaluation is more robust but harder to standardize.
- **Startup overhead:** Running a full T0-T6 ablation is itself expensive. For low-stakes tasks, use heuristic tier selection (trivial = T0, moderate = T1, complex = T5) rather than running full ablation.

## Reference

Villmow, M. (2026). *Taming Scylla: Understanding the multi-headed agentic daemon of the coding seas.* arXiv:2602.08765v1. [https://arxiv.org/abs/2602.08765v1](https://arxiv.org/abs/2602.08765v1)

Key takeaway: Section 4 (Ablation Results) contains the tier-by-tier CoP breakdowns and the Token Efficiency Chasm analysis. Section 5 discusses the T5 hybrid construction method. Table 3 shows the multi-judge scoring rubric weights.