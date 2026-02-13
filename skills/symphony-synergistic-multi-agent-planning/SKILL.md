---
name: "symphony-synergistic-multi-agent-planning"
description: >
  Orchestrate heterogeneous multi-agent MCTS planning for complex reasoning and search tasks.
  Uses a pool of diverse LLM-based agents with distinct roles (planner, executor, verifier, reflector)
  to generate diverse solution branches via Monte Carlo Tree Search, producing more robust plans
  than any single-agent approach. Trigger phrases: "use SYMPHONY planning", "multi-agent MCTS",
  "heterogeneous agent planning", "diverse rollout search", "synergistic planning agents",
  "multi-model tree search"
---

# SYMPHONY: Synergistic Multi-Agent Planning with Heterogeneous Language Model Assembly

This skill enables Claude to orchestrate a **heterogeneous pool of specialized agents** that collaboratively explore solution spaces using Monte Carlo Tree Search (MCTS). Instead of relying on a single model to both generate candidate actions and evaluate them -- which collapses diversity and gets stuck in local optima -- SYMPHONY assigns distinct agent roles (planner, executor, verifier, reflector) with varied reasoning strategies, temperatures, and prompts. Each agent contributes independent rollouts at tree nodes, and their evaluations are aggregated to produce more calibrated value estimates. The result is systematically better exploration for multi-step reasoning, code generation, multi-hop QA, and sequential decision-making tasks.

## When to Use

- When the user asks to solve a **multi-step planning problem** (e.g., "plan the steps to migrate this monolith to microservices")
- When a task requires **exploring multiple solution paths** before committing (e.g., "find the best refactoring strategy for this module")
- When the user wants to **compare diverse approaches** to a code architecture or algorithm design problem
- When solving **multi-hop reasoning** tasks where a single chain-of-thought frequently fails (e.g., complex debugging across multiple files)
- When the user explicitly requests "multi-agent search", "MCTS planning", or "SYMPHONY-style exploration"
- When a sequential decision task benefits from **rollout diversity** -- e.g., generating and evaluating multiple API integration strategies, migration paths, or deployment plans

## Key Technique

**The core problem:** Single-agent MCTS uses one LLM to both propose actions (expansion) and evaluate outcomes (simulation/reward). This produces homogeneous rollouts -- the same model reasons the same way repeatedly, limiting branch diversity and causing the search tree to converge prematurely on suboptimal solutions.

**SYMPHONY's solution:** Assemble a pool of 4-8 heterogeneous agents, each with a distinct role and reasoning configuration. During MCTS rollouts, different agents generate candidate actions and evaluate trajectories independently. Their reward signals are aggregated (weighted by historical reliability) during backpropagation. This produces genuine diversity: a "planner" agent with high temperature explores creative options, a "verifier" agent with low temperature rigorously checks constraints, a "reflector" agent critiques partial solutions, and an "executor" agent tests concrete implementations. The heterogeneity is what breaks the single-model echo chamber.

**Why it works in practice:** Even when all agents are powered by the same underlying model (e.g., Claude), varying their system prompts, roles, temperature settings, and evaluation criteria produces meaningfully different reasoning trajectories. The UCT (Upper Confidence bound for Trees) selection policy balances exploring under-visited branches against exploiting high-reward paths, while multi-agent aggregation produces more robust value estimates than any single evaluator. The paper shows this approach outperforms single-agent baselines on HotpotQA, WebShop, MBPP, and custom planning benchmarks.

## Step-by-Step Workflow

1. **Decompose the problem into a search tree structure.** Identify the root state (current situation), the action space at each node (candidate decisions/steps), and the terminal condition (what constitutes a complete solution). Frame the user's task as: "Starting from state S, find a sequence of actions leading to goal G."

2. **Define the heterogeneous agent pool.** Create 4-6 agents with distinct roles:
   - **Planner**: High creativity (temperature ~0.9). Generates diverse candidate actions at expansion nodes. System prompt emphasizes brainstorming and lateral thinking.
   - **Executor**: Medium temperature (~0.5). Simulates executing a candidate action -- writes code, drafts text, or traces logic to see what happens next.
   - **Verifier**: Low temperature (~0.2). Evaluates whether a rollout trajectory satisfies constraints, catches bugs, checks logical consistency.
   - **Reflector**: Medium temperature (~0.6). Critiques partial solutions, identifies gaps, suggests course corrections.
   - (Optional) **Domain Specialist**: Configured with domain-specific context or few-shot examples relevant to the task.

3. **Initialize the MCTS tree.** Create the root node representing the current problem state. Set the exploration constant (C = sqrt(2) is standard) for UCT selection.

4. **Selection phase.** Starting from the root, descend the tree by selecting child nodes using UCT: `UCT(node) = Q(node)/N(node) + C * sqrt(ln(N(parent)) / N(node))`, where Q is accumulated reward and N is visit count. This balances exploitation of known-good paths with exploration of less-visited alternatives.

5. **Expansion phase.** At the selected leaf node, use the **Planner** agent to generate 2-4 candidate next actions. Each becomes a new child node. The Planner's high temperature ensures diverse candidates rather than repetitive near-duplicates.

6. **Simulation phase (multi-agent rollout).** For each new child node, run rollouts using **different agents**:
   - The **Executor** traces the action forward to a terminal state (or depth limit).
   - The **Verifier** scores the resulting trajectory on correctness, feasibility, and constraint satisfaction.
   - The **Reflector** provides qualitative assessment of trajectory quality and suggests whether this direction is promising.

7. **Reward aggregation.** Combine agent evaluations into a single reward signal: `R = w_executor * R_executor + w_verifier * R_verifier + w_reflector * R_reflector`. Weights reflect agent reliability (verifier typically gets highest weight for correctness-critical tasks). Normalize to [0, 1].

8. **Backpropagation.** Propagate the aggregated reward up the tree path, updating Q and N values at each ancestor node.

9. **Iterate until budget exhausted.** Repeat steps 4-8 for a fixed number of iterations (typically 10-50 for practical coding tasks). More iterations for higher-stakes decisions.

10. **Extract the best plan.** Follow the path from root to leaf with the highest average reward (Q/N). Present this as the recommended solution, along with the runner-up path for comparison. Include the Verifier's assessment of the chosen path.

## Concrete Examples

**Example 1: Choosing a database migration strategy**

```
User: "I need to migrate our PostgreSQL database from a monolithic schema to
a multi-tenant architecture. What's the best approach?"

Approach:
1. Root state: Current monolithic schema with shared tables.
   Action space: {schema-per-tenant, row-level-security, separate-databases, hybrid}.
   Terminal: Complete migration plan with rollback strategy.

2. Agent pool:
   - Planner: Generates 4 candidate strategies with creative variants
   - Executor: Traces each strategy through implementation steps,
     estimating migration scripts, downtime, and data movement
   - Verifier: Checks each strategy against constraints (zero-downtime
     requirement, data isolation, query performance)
   - Reflector: Identifies hidden risks (e.g., cross-tenant queries
     breaking, connection pool exhaustion)

3. MCTS iterations (15 rounds):
   - Round 1-5: Explores all 4 strategies broadly
   - Round 6-10: Focuses on row-level-security and hybrid (highest UCT)
   - Round 11-15: Deep-dives into hybrid approach with RLS for hot
     tables and schema separation for cold tables

Output:
   Recommended plan: Hybrid approach
   - Phase 1: Add tenant_id column + RLS policies to high-traffic tables
   - Phase 2: Migrate cold/archival tables to per-tenant schemas
   - Phase 3: Update application connection routing
   - Rollback: RLS policies are additive; revert by dropping policies
   Verifier confidence: 0.85 (concern: RLS performance on tables >100M rows)
   Runner-up: Full schema-per-tenant (higher isolation, but 3x migration effort)
```

**Example 2: Debugging a complex distributed system issue**

```
User: "Our microservice is intermittently returning 500 errors under load.
Logs show timeout exceptions but the downstream service is healthy."

Approach:
1. Root state: Intermittent 500s with timeout exceptions.
   Action space: {connection-pool-exhaustion, DNS-resolution-delay,
   circuit-breaker-misconfiguration, thread-starvation, GC-pauses}.

2. Agent pool:
   - Planner: Proposes 5 hypotheses as candidate root causes
   - Executor: For each hypothesis, traces the causal chain and
     identifies what evidence would confirm/deny it
   - Verifier: Cross-checks each hypothesis against the stated symptoms
     (intermittent, load-dependent, downstream healthy)
   - Reflector: Identifies which hypotheses are mutually exclusive vs.
     potentially compounding

3. MCTS iterations (12 rounds):
   - Connection-pool-exhaustion scores highest after Verifier confirms
     it explains load-dependence + healthy downstream
   - Thread-starvation is runner-up (similar symptoms)

Output:
   Most likely root cause: Connection pool exhaustion
   Diagnostic steps:
   1. Check connection pool metrics: `SELECT count(*) FROM pg_stat_activity`
   2. Review pool config: max connections, idle timeout, connection lifetime
   3. Look for connection leaks: unclosed connections in error paths
   4. Load test with pool monitoring enabled

   Alternative hypothesis: Thread starvation in async executor
   - Check: Thread pool size vs. concurrent request count
```

**Example 3: Designing an API versioning strategy**

```
User: "We need to version our REST API. We have 50+ endpoints and 3 major
client apps. What's the right approach?"

Approach:
1. Root: Unversioned API with 50+ endpoints, 3 clients.
   Actions: {URL-path-versioning, header-versioning, query-param-versioning,
   content-negotiation, GraphQL-migration}.

2. MCTS with 4 agents explores each strategy:
   - Planner generates variants (e.g., URL versioning with /v2 prefix
     vs. per-resource versioning)
   - Executor traces implementation for each: routing changes, client
     SDK updates, documentation
   - Verifier checks: backward compatibility, client migration burden,
     cacheability, API gateway support
   - Reflector flags: "Header versioning breaks CDN caching",
     "GraphQL migration is scope creep for this ask"

Output:
   Recommended: URL path versioning (/v1/, /v2/)
   - Simplest client migration (update base URL)
   - Full CDN/proxy compatibility
   - Clear deprecation timeline per version
   Implementation:
   1. Add version prefix to router: /api/v1/*
   2. Create v2 route group, initially proxying to v1 handlers
   3. Migrate endpoints incrementally, client by client
   4. Set v1 sunset header after all clients on v2

   Verifier score: 0.91 (minor concern: URL aesthetics)
```

## Best Practices

- **Do:** Assign genuinely different system prompts and temperatures to each agent. The power of SYMPHONY comes from heterogeneity -- if all agents reason identically, you get no diversity benefit.
- **Do:** Weight the Verifier agent's reward signal highest for correctness-critical tasks (code generation, security decisions). Weight the Planner highest for creative/design tasks.
- **Do:** Set a reasonable iteration budget. For most coding tasks, 10-20 MCTS iterations with 4 agents is sufficient. Diminishing returns set in quickly.
- **Do:** Present the top 2 paths to the user, not just the winner. The runner-up often contains valuable insights or serves as a fallback.
- **Avoid:** Using identical prompts for all agents -- this defeats the purpose and produces the same single-agent homogeneity the technique is designed to overcome.
- **Avoid:** Over-expanding the tree. Limit branching factor to 2-4 children per node. More branches spread rollout budget too thin, producing noisy value estimates.
- **Avoid:** Skipping the Reflector agent. Without critique, the search tends to commit to the first "good enough" path rather than finding the best one.

## Error Handling

- **Agent disagreement (no consensus):** If the Verifier and Executor give opposite signals on a node, increase rollout count for that subtree rather than averaging away the conflict. Flag the disagreement to the user.
- **All branches score poorly:** If no path exceeds a reward threshold of 0.3 after half the iteration budget, stop and ask the user for additional constraints or context. The problem may be under-specified.
- **Rollout depth exceeded:** If simulation reaches the depth limit without a terminal state, use the Verifier's partial-trajectory score as a heuristic reward (discounted by remaining depth).
- **Degenerate action space:** If the Planner generates near-duplicate actions at a node, re-prompt with explicit diversity instructions ("generate an approach that is fundamentally different from: [list existing children]").

## Limitations

- **Latency cost:** Multi-agent rollouts multiply inference calls. For time-sensitive tasks where a quick single-pass answer suffices, SYMPHONY is overkill.
- **Shallow action spaces:** If a problem has only 2-3 obvious next actions (e.g., "fix this typo"), tree search adds no value. Use SYMPHONY for genuinely branching decision spaces.
- **Evaluation difficulty:** The technique requires meaningful reward signals. If you cannot define what "better" means for a trajectory (highly subjective tasks like prose style), reward aggregation becomes unreliable.
- **Single-model diversity ceiling:** When all agents share the same underlying model, prompt/temperature variation helps but cannot fully replicate the diversity of truly different model architectures. For maximum benefit, mix model families when possible.
- **Not for simple tasks:** A straightforward bug fix or small feature does not warrant MCTS exploration. Reserve this for architectural decisions, complex debugging, and multi-step planning where the cost of choosing the wrong path is high.

## Reference

**Paper:** [SYMPHONY: Synergistic Multi-agent Planning with Heterogeneous Language Model Assembly](https://arxiv.org/abs/2601.22623v1) (Zhu, Tang, Yue -- NeurIPS 2025)
**Key takeaway:** Heterogeneous agent pools with role-differentiated prompts and temperatures produce meaningfully diverse MCTS rollouts, and multi-agent reward aggregation yields more calibrated value estimates than single-model evaluation -- even when all agents share the same base model.