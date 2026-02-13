---
name: "metagen-self-evolving-roles-topologies"
description: |
  Self-evolving multi-agent orchestration that dynamically generates specialized roles and collaboration
  topologies at inference time. Instead of fixed agent roles, MetaGen creates query-conditioned role
  specifications, builds a minimal execution DAG, and iteratively refines both roles and structure using
  lightweight feedback signals. Use when:
  - "Set up a multi-agent pipeline to solve this problem"
  - "Use self-evolving agents to generate and debug this code"
  - "Orchestrate multiple specialized agents for this reasoning task"
  - "Dynamically create roles to handle this complex task"
  - "Build an adaptive agent topology for this multi-step problem"
  - "Use MetaGen-style agent collaboration"
---

# MetaGen: Self-Evolving Roles and Topologies for Multi-Agent Reasoning

This skill enables Claude to orchestrate multi-agent reasoning using the MetaGen framework: dynamically generating task-specific agent roles, wiring them into a minimal directed acyclic graph (DAG), and iteratively refining both the roles and the topology based on execution feedback. Instead of relying on a fixed set of agents with predetermined interactions, MetaGen adapts the entire collaboration structure to each query — producing better results with fewer tokens than static multi-agent designs.

## When to Use

- When a user asks you to solve a complex problem that benefits from multiple specialized perspectives (code generation + review + testing, multi-step math reasoning, cross-domain analysis)
- When the user explicitly requests multi-agent collaboration, agent orchestration, or role-based decomposition
- When a task has failed with a single-pass approach and needs iterative refinement from structurally different viewpoints
- When the problem domain is unfamiliar or ambiguous, requiring the system to discover what expertise is needed rather than assuming it upfront
- When the user wants to minimize token cost while maintaining accuracy across a batch of heterogeneous tasks
- When tackling a non-stationary stream of tasks where the required expertise shifts (e.g., alternating between code, math, and NLI problems)

## Key Technique

**The core insight of MetaGen is that both _who_ collaborates and _how_ they collaborate should be determined by the query, not predetermined by the developer.** Traditional multi-agent systems pick from a fixed role library (e.g., "programmer", "reviewer", "tester") and wire them in a fixed topology (chain, star, debate). MetaGen replaces both with dynamic generation: an Architect agent analyzes the query, proposes candidate roles as structured tuples (name, description, system prompt, user prompt, capabilities), filters them for validity and semantic novelty, then selects a committee using epsilon-greedy exploration over learned priority scores.

**The execution topology is a constrained DAG built around a minimal skeleton.** For code generation, the skeleton might be `hub -> programmer -> evaluator`. MetaGen scores candidate edges using feature vectors that combine lexical cues, capability indicators, semantic relevance, and historical statistics, then prunes to a minimal graph that still connects input to output. This avoids the token explosion of fully-connected or tree topologies — MetaGen uses 85% fewer tokens than comparable dynamic-topology systems.

**Refinement happens at two timescales.** _Intra-task_: during a single execution, feedback signals (compilation errors, failed tests, format violations, self-consistency checks) trigger role prompt rewrites for underperforming agents and conservative edge modifications (swapping or deactivating connections while preserving DAG structure). _Inter-task_: across instances, a scalar reward (correctness minus token cost) updates linear selection priors via a reward-weighted rule, and successful roles are solidified into a persistent cache for reuse on future queries.

## Step-by-Step Workflow

1. **Analyze the query to identify required expertise domains.** Read the user's request and determine what distinct competencies are needed (e.g., algorithm design, edge-case analysis, test writing, performance optimization). Do not assume a fixed set — let the problem dictate the roles.

2. **Generate candidate role specifications as structured tuples.** For each identified competency, produce a role with: a descriptive name, a one-sentence description, a system prompt defining the agent's perspective and constraints, a user prompt template with placeholders for the query and prior context, and a capability list. Aim for 3-6 candidates.

3. **Filter candidates for validity and novelty.** Discard any role whose prompt template is missing required placeholders or contains restricted/redundant instructions. Among remaining candidates, ensure semantic diversity — if two roles overlap substantially in their described capabilities, merge or drop one. Keep the pool lean: 2-4 active roles for most tasks.

4. **Select the working committee using epsilon-greedy exploration.** Rank candidates by estimated utility for this specific query (relevance of capabilities to the problem). With probability epsilon (~0.1), substitute a lower-ranked candidate to explore new configurations. This prevents the system from collapsing to a single favorite configuration.

5. **Construct a minimal execution DAG around a task-type skeleton.** Define the backbone flow (e.g., `coordinator -> specialist_1 -> specialist_2 -> evaluator`). Score potential additional edges by estimated information value. Add only edges that exceed a utility threshold. Enforce acyclicity — no agent's output should feed back to an upstream agent in the same iteration.

6. **Execute the DAG, collecting feedback signals at each node.** Run each agent in topological order, passing upstream outputs as context. At each node, capture observable signals: does the output parse correctly? Does generated code compile? Do assertions pass? Are there internal contradictions?

7. **Rewrite underperforming role prompts using feedback traces.** If a node produces output that triggers error signals, revise its system/user prompt to address the specific failure mode. For example, if a code-generating agent produces syntax errors, add explicit formatting constraints to its prompt. Do not change well-performing roles.

8. **Adjust topology edges conservatively.** If a role consistently produces unhelpful output despite prompt revision, deactivate its incoming/outgoing edges and either swap in an alternative role from the candidate pool or promote a new candidate. Preserve the path from entry to exit node. Maintain DAG structure.

9. **Aggregate final outputs through the evaluator/exit node.** The terminal agent synthesizes outputs from all upstream paths, resolves conflicts, and produces the final answer. If the evaluator detects remaining issues, trigger one more refinement cycle (up to a maximum of 2-3 iterations to bound cost).

10. **Update priors for future queries.** Record which role configurations and topology structures led to successful outcomes. Compute a scalar reward (correctness indicator minus a token-cost penalty). Use this to bias future role selection toward proven configurations while maintaining exploration.

## Concrete Examples

**Example 1: Complex Code Generation with Edge Cases**

```
User: Write a Python function that parses cron expressions into human-readable
descriptions, handling all standard fields including day-of-week names and
month names.

Approach:
1. Analyze query -> needs: algorithm design, string parsing expertise,
   edge-case reasoning, test generation.
2. Generate roles:
   - Architect: decomposes the cron spec into subproblems per field
   - Parser-Specialist: writes the core parsing logic for each field type
   - Edge-Case-Analyst: identifies tricky inputs (wildcards, ranges, steps,
     combined expressions like "1,15" or "MON-FRI")
   - Test-Engineer: produces comprehensive test cases
3. Skeleton DAG: Architect -> Parser-Specialist -> Edge-Case-Analyst -> Test-Engineer
   Additional edge: Edge-Case-Analyst feeds back constraints to Parser-Specialist
   (within one refinement cycle, not a loop).
4. Execute:
   - Architect outputs field decomposition and interface contract
   - Parser-Specialist writes initial implementation
   - Edge-Case-Analyst reviews, finds "*/5" step syntax and "SUN-SAT" name
     mapping are underhandled -> feedback triggers prompt rewrite for Parser
   - Parser-Specialist revises with explicit step and name-mapping logic
   - Test-Engineer generates 15 test cases covering all field types
5. Evaluator runs tests, confirms 15/15 pass, returns final implementation.

Output: A tested, robust cron parser function with clear docstring.
```

**Example 2: Multi-Step Mathematical Reasoning**

```
User: Solve this: A factory produces widgets in batches. Each batch has a 3%
defect rate. A quality inspector samples 5 widgets from each batch. What is
the probability that the inspector finds exactly 1 defective widget, and how
does this change if the batch size is 50 vs 500?

Approach:
1. Analyze query -> needs: probability modeling, hypergeometric vs binomial
   distinction, numerical computation, explanation clarity.
2. Generate roles:
   - Problem-Formulator: identifies the correct probability distribution
     and states assumptions explicitly
   - Calculator: performs the numerical computation for both batch sizes
   - Assumption-Checker: evaluates whether binomial approximation is valid
     for each batch size or if hypergeometric is needed
3. Skeleton DAG: Problem-Formulator -> Calculator, Problem-Formulator ->
   Assumption-Checker, both feed into Synthesizer.
4. Execute:
   - Problem-Formulator identifies this as hypergeometric (sampling without
     replacement from finite batch), notes binomial approximation validity
     depends on batch-to-sample ratio
   - Calculator computes exact hypergeometric probabilities for both sizes
   - Assumption-Checker flags that batch=50 has meaningful finite-population
     correction (sample is 10% of batch) while batch=500 is well-approximated
     by binomial
   - Synthesizer combines: gives exact answers, explains the difference,
     shows when the approximation breaks down
5. No feedback-triggered revisions needed — all signals clean.

Output: Exact probabilities for both cases with explanation of why they differ
and when binomial approximation is appropriate.
```

**Example 3: Debugging a Flaky Integration Test**

```
User: Our CI test `test_concurrent_db_writes` passes locally but fails
intermittently in CI. Here's the test code and the error trace.

Approach:
1. Analyze query -> needs: concurrency analysis, test infrastructure
   knowledge, race condition detection, fix proposal.
2. Generate roles:
   - Concurrency-Analyst: examines shared state, locking, timing assumptions
   - CI-Environment-Specialist: identifies differences between local and CI
     (CPU count, disk speed, connection pooling)
   - Fix-Proposer: generates concrete code changes
3. Skeleton: Concurrency-Analyst + CI-Environment-Specialist (parallel) ->
   Fix-Proposer -> Evaluator.
4. Execute:
   - Concurrency-Analyst identifies a race condition: two goroutines write
     to the same row without serialization, succeeding locally due to faster
     disk but failing under CI's shared-resource contention
   - CI-Environment-Specialist confirms: CI uses shared Postgres with
     connection pooling, increasing lock contention probability
   - Fix-Proposer drafts two options: (a) add SELECT FOR UPDATE, (b) use
     advisory locks
   - Evaluator: option (a) is simpler and sufficient -> selects it
5. Feedback: if the fix introduces a deadlock risk (detected by static
   analysis), rewrite Fix-Proposer prompt to add deadlock-avoidance
   constraint, re-execute.

Output: Root cause explanation + targeted fix with rationale for the choice.
```

## Best Practices

- **Do:** Start with the minimum viable topology (2-3 roles, linear DAG) and add complexity only when feedback signals indicate a gap. MetaGen's power is in targeted adaptation, not in spawning many agents upfront.
- **Do:** Define each role's system prompt with specific constraints and output format expectations. Vague roles like "helper" or "assistant" defeat the purpose of specialization.
- **Do:** Use observable, automatic feedback signals (compilation results, test outcomes, format validation) rather than subjective quality judgments for triggering refinement cycles.
- **Do:** Maintain a cache of role configurations that worked well for similar past queries. Reuse proven roles as starting points and adapt from there.
- **Avoid:** Fully-connected topologies where every agent sees every other agent's output. This explodes token cost with marginal accuracy gains. Keep the DAG sparse.
- **Avoid:** More than 2-3 refinement iterations per task. Diminishing returns set in quickly, and unbounded iteration negates the cost savings. If the output is still failing after 3 cycles, the role decomposition itself needs rethinking, not more iterations.
- **Avoid:** Generating roles that duplicate the same capability with different names. Enforce semantic novelty — if two roles have overlapping capability descriptions, merge them or drop the weaker one.

## Error Handling

| Failure Mode | Detection Signal | Recovery Action |
|---|---|---|
| Generated role has invalid/incomplete prompt template | Missing placeholders, schema validation failure | Regenerate the role with explicit template requirements; fall back to a known-good role from cache |
| DAG becomes disconnected (no path from entry to exit) | Graph connectivity check after edge modification | Restore the last connected topology; only retry the specific edge change that caused disconnection |
| Agent produces output that downstream agents cannot parse | Format validation error at receiving node | Rewrite the producing agent's prompt to enforce the expected output schema; add an explicit format example |
| Refinement loop does not converge (oscillating rewrites) | Same error class recurring after 2 prompt rewrites | Freeze the current best output, swap the failing role for an alternative candidate, or simplify the topology |
| Token budget exceeded before completion | Running token counter exceeds threshold | Collapse remaining topology to a single-agent direct-answer path using accumulated context |

## Limitations

- **Single-model ceiling:** MetaGen does not change the underlying model — all roles run on the same LLM. If the base model lacks a capability (e.g., advanced mathematical reasoning), no amount of role specialization will compensate. The technique amplifies existing capabilities through structure, not magic.
- **Overhead on simple tasks:** For straightforward, single-step questions, the role generation and topology construction overhead is pure waste. Only use MetaGen-style orchestration when the task genuinely benefits from multiple perspectives or iterative refinement.
- **Feedback signal dependency:** The iterative refinement loop is only as good as the feedback signals available. Tasks without clear success criteria (creative writing, subjective design) get limited benefit from the intra-task evolution mechanism.
- **Cold start on novel domains:** Without cached role priors, the first query in an entirely new domain relies on the Architect's generation quality. Expect the first attempt to be exploratory; performance improves as the role cache accumulates successful configurations.
- **Acyclicity constraint:** The DAG structure prevents circular deliberation patterns (e.g., debate) that some tasks benefit from. For tasks where iterative debate between peers is valuable, MetaGen's topology is suboptimal compared to debate-style frameworks.

## Reference

**Paper:** [MetaGen: Self-Evolving Roles and Topologies for Multi-Agent LLM Reasoning](https://arxiv.org/abs/2601.19290v1) — Wang et al., 2026. Key sections: Algorithm 1 (full inference-time procedure), Section 3.1 (generative role space with constraint filtering and embedding-based diversity gating), Section 3.3 (intra-task evolution via prompt rewriting and prior-filtered edge exploration), Table 1 (benchmark results showing 95.1% average accuracy with 85% token reduction vs. G-Designer).