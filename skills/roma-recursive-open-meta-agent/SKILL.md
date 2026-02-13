---
name: "roma-recursive-open-meta-agent"
description: "Decompose long-horizon, multi-step tasks using ROMA's recursive meta-agent pattern: Atomizer decides if a task needs splitting, Planner builds a dependency-aware subtask DAG, Executors run leaf tasks in parallel, and Aggregator compresses results bottom-up. Use when the user says 'break this project into subtasks', 'plan a complex multi-step workflow', 'decompose this into parallel work', 'use recursive task decomposition', 'orchestrate agents for this large task', or 'ROMA-style planning'."
---

# ROMA: Recursive Open Meta-Agent Framework

This skill enables Claude to decompose complex, long-horizon tasks into dependency-aware subtask trees and orchestrate their execution using the ROMA pattern. Instead of attempting large goals in a single sequential pass (which causes context window degradation and brittle reasoning chains), ROMA applies a uniform recursive control loop at every node: **Atomize** (decide if the task is atomic), **Plan** (decompose into a DAG of subtasks), **Execute** (run leaf tasks with type-specialized strategies), and **Aggregate** (compress and validate child results before passing them upward). This keeps each agent's working context small, enables parallel execution of independent branches, and produces transparent hierarchical traces that make failures easy to localize.

## When to Use

- When the user asks to build a full-stack feature spanning frontend, backend, database, and tests -- tasks that touch 5+ files across multiple concerns.
- When the user says "break this down" or "plan this project" for a task that has clear subtask dependencies (e.g., "build an auth system with login, signup, password reset, and session management").
- When orchestrating a multi-agent swarm where different agents handle research, coding, testing, and documentation in parallel.
- When a task has failed partway through sequential execution and needs to be restructured into isolated, retryable units.
- When the user wants to parallelize independent work items (e.g., "refactor these 8 API endpoints") while respecting shared dependencies.
- When context window pressure is a concern -- the task generates so much intermediate output that a single-pass approach would degrade quality.

## Key Technique

**Recursive Decomposition with Dependency-Aware DAGs.** ROMA's core insight is that a single recursive control loop -- Atomize, Plan, Execute, Aggregate -- applied uniformly at every level of the task tree eliminates the need for bespoke orchestration logic. The Planner produces a directed acyclic graph (DAG) of subtasks with explicit dependency edges. Independent subtasks execute in parallel; dependent subtasks block until their prerequisites complete. This is not simple "divide and conquer" -- the DAG structure captures real data and ordering dependencies (e.g., "database migration must complete before API endpoint implementation; both can proceed before frontend work begins").

**Bounded Context Through Aggregation.** The critical failure mode of sequential multi-step agents is context rot: intermediate results accumulate in the prompt until the model's reasoning degrades. ROMA's Aggregator solves this by performing three operations at each tree node: (1) **synthesis** -- combining child outputs into a parent-scoped format, (2) **verification** -- checking consistency of child results, and (3) **compression** -- reducing artifacts to concise summaries before propagating them upward. Each executor works on an isolated context containing only the information relevant to its leaf task, not the entire conversation history.

**Type-Specialized Execution.** ROMA classifies leaf tasks into four types -- **search** (retrieval/research), **think** (reasoning/analysis), **write** (composition/documentation), and **code** (implementation/transformation) -- each routed to an executor with a distinct prompting strategy (ReAct for search, Chain-of-Thought for think, structured generation for write, CodeAct for code). This prevents the "jack of all trades" problem where a single generic prompt handles every task type poorly.

## Step-by-Step Workflow

1. **Receive the goal and run the Atomizer check.** Evaluate whether the task can be completed in a single, focused action (atomic) or requires decomposition. Criteria: Does it touch multiple files? Does it have sequential dependencies? Would a single attempt exceed reasonable context? If atomic, execute directly and return the result.

2. **Decompose with the Planner into a MECE subtask DAG.** Break the goal into subtasks that are Mutually Exclusive (no overlapping work) and Collectively Exhaustive (full coverage of the goal). For each subtask, specify: a short title, the task type (`search`, `think`, `write`, or `code`), a clear description of the expected input/output, and explicit dependency edges to other subtasks.

3. **Render the DAG as a structured plan.** Output the subtask tree as a numbered list or markdown table showing: task ID, title, type, dependencies (by ID), and status. This becomes the transparent execution trace. Use the TodoWrite tool to track each subtask.

4. **Identify the critical path and parallelizable branches.** Topologically sort the DAG. Group tasks with no unmet dependencies into parallel execution batches. The critical path (longest dependency chain) determines minimum total time.

5. **Execute leaf tasks with type-specialized strategies.** For each ready-to-execute atomic task:
   - `search`: Use web search, file search, or grep to retrieve information. Return structured findings.
   - `think`: Apply chain-of-thought reasoning. Return analysis or decision with justification.
   - `write`: Generate structured prose, documentation, or config. Return the artifact.
   - `code`: Write, edit, or run code. Return the implementation and any test results.

6. **Aggregate child results at each parent node.** When all children of a node complete, synthesize their outputs into a parent-scoped result. Verify consistency (e.g., does the API endpoint match the database schema the migration created?). Compress the result -- strip intermediate reasoning, keep only the deliverable and key decisions.

7. **Propagate aggregated results upward.** Pass the compressed parent result to the next level. The grandparent node sees only the aggregated summary, not the raw child outputs. This keeps context bounded at every level.

8. **Handle failures locally.** If a leaf task fails, retry it in isolation with its specific context -- do not re-run the entire tree. If a subtask produces inconsistent results during aggregation, flag the specific child for re-execution. Update the DAG status to reflect partial completion.

9. **Deliver the final aggregated result.** The root node's aggregation produces the final deliverable. Include a summary execution trace showing which subtasks ran, their types, and their completion status.

## Concrete Examples

**Example 1: Building an Authentication System**

```
User: Add email/password authentication to our Express app with signup,
login, password reset, and session management.

Approach (ROMA Decomposition):

Atomizer: NOT atomic -- 4 features, database changes, middleware, routes, tests.

Planner DAG:
  ID | Task                          | Type  | Depends On | Status
  1  | Research existing auth setup   | search | --        | pending
  2  | Design auth database schema    | think  | 1         | pending
  3  | Create user migration & model  | code   | 2         | pending
  4  | Implement signup endpoint      | code   | 3         | pending
  5  | Implement login endpoint       | code   | 3         | pending
  6  | Implement password reset flow  | code   | 3         | pending
  7  | Add session middleware         | code   | 3         | pending
  8  | Write integration tests        | code   | 4,5,6,7   | pending
  9  | Write API documentation        | write  | 4,5,6,7   | pending

Parallel batches:
  Batch 1: [1]
  Batch 2: [2]
  Batch 3: [3]
  Batch 4: [4, 5, 6, 7]  -- all independent, share only the User model
  Batch 5: [8, 9]         -- both depend on all endpoints being done

Aggregation at root: Verify that signup creates users the login endpoint
can authenticate, that password reset tokens reference valid users, and
that session middleware protects all endpoints. Compress into a summary
of files changed and how to test.
```

**Example 2: Refactoring 6 API Endpoints from REST to GraphQL**

```
User: Migrate our REST endpoints (users, posts, comments, tags, media,
notifications) to GraphQL resolvers.

Approach (ROMA Decomposition):

Atomizer: NOT atomic -- 6 independent migrations with a shared schema setup.

Planner DAG:
  ID | Task                            | Type   | Depends On
  1  | Analyze existing REST endpoints  | search | --
  2  | Design unified GraphQL schema    | think  | 1
  3  | Set up GraphQL server config     | code   | 2
  4  | Migrate users endpoint           | code   | 3
  5  | Migrate posts endpoint           | code   | 3
  6  | Migrate comments endpoint        | code   | 3
  7  | Migrate tags endpoint            | code   | 3
  8  | Migrate media endpoint           | code   | 3
  9  | Migrate notifications endpoint   | code   | 3
  10 | Integration tests for all        | code   | 4,5,6,7,8,9

Parallel batches:
  Batch 1: [1]
  Batch 2: [2]
  Batch 3: [3]
  Batch 4: [4, 5, 6, 7, 8, 9]  -- all independent after schema is set
  Batch 5: [10]

Aggregation: Each resolver migration returns the resolver file and its
type definitions. Parent aggregator merges type defs into the unified
schema, checks for naming conflicts, and verifies cross-references
(e.g., Post.author resolves to User type correctly).
```

**Example 3: Research and Write a Technical Report**

```
User: Research the tradeoffs between SQLite, PostgreSQL, and DynamoDB
for our IoT sensor data pipeline and write a recommendation report.

Approach (ROMA Decomposition):

Atomizer: NOT atomic -- requires research, analysis, and composition.

Planner DAG:
  ID | Task                                  | Type   | Depends On
  1  | Analyze current data pipeline needs    | search | --
  2  | Research SQLite for IoT workloads      | search | --
  3  | Research PostgreSQL for IoT workloads  | search | --
  4  | Research DynamoDB for IoT workloads    | search | --
  5  | Compare on latency, cost, scalability  | think  | 1,2,3,4
  6  | Draft recommendation with tradeoffs    | write  | 5
  7  | Write migration plan for top choice    | write  | 5

Parallel batches:
  Batch 1: [1, 2, 3, 4]  -- all research runs concurrently
  Batch 2: [5]
  Batch 3: [6, 7]         -- report and migration plan are independent

Aggregation: Verify the recommendation in task 6 is consistent with the
comparison in task 5. Merge tasks 6 and 7 into a single deliverable
document with executive summary, comparison table, recommendation, and
migration steps.
```

## Best Practices

- **Do:** Make dependency edges explicit. Every subtask must declare what it depends on by ID. Implicit ordering leads to race conditions and missing context.
- **Do:** Classify every leaf task by type (`search`, `think`, `write`, `code`). This drives executor strategy selection and produces better results than generic execution.
- **Do:** Aggregate aggressively. Each parent should receive a compressed summary, not raw child output. If a child task produced 200 lines of code, the aggregated result for the parent is "created `auth.service.ts` with signup/login/reset methods, exports `AuthService` class."
- **Do:** Keep leaf tasks small enough that each can succeed in a single focused execution. If an "atomic" task still feels large, apply the Atomizer check recursively.
- **Avoid:** Skipping the Atomizer check. Not every task needs decomposition. Forcing ROMA on a simple "fix this typo" task adds overhead with no benefit.
- **Avoid:** Creating subtasks with circular dependencies. The DAG must be acyclic. If task A needs B's output and B needs A's output, restructure: extract the shared dependency into a third task C that both depend on.
- **Avoid:** Propagating raw intermediate output upward. This defeats ROMA's context management. Always compress at aggregation boundaries.

## Error Handling

| Failure Mode | Detection | Recovery |
|---|---|---|
| Leaf task fails (code won't compile, search returns nothing) | Executor reports error | Retry the specific leaf with refined instructions. Do not re-run siblings. |
| Aggregation inconsistency (child results contradict each other) | Aggregator verification step | Flag conflicting children. Re-execute the one with lower confidence or ask the user to resolve the ambiguity. |
| Decomposition too shallow (a "leaf" is still too complex) | Executor struggles or produces partial result | Re-apply the Atomizer: decompose the failing leaf into its own subtask DAG. This is the recursive case. |
| Decomposition too deep (trivial tasks split unnecessarily) | Many single-line leaf tasks with no real dependencies | Collapse trivial sibling tasks back into their parent. Execute as a batch. |
| Dependency cycle detected | Topological sort fails | Restructure the DAG: identify the circular dependency, extract shared state into a prerequisite task. |

## Limitations

- **Overhead on small tasks.** ROMA's four-role loop (Atomize, Plan, Execute, Aggregate) adds planning and aggregation cost. For tasks completable in under 3 steps, skip ROMA and execute directly.
- **Decomposition quality depends on domain understanding.** The Planner must understand the problem domain well enough to produce MECE subtasks. Ambiguous or novel domains may produce poor decompositions that need user correction.
- **Aggregation is lossy by design.** Compressing child results means some detail is discarded. If downstream tasks need fine-grained details from earlier steps, the dependency edges must be explicit so the raw output is available, not just the aggregated summary.
- **Not a replacement for domain-specific agents.** ROMA is an orchestration pattern, not a domain solver. The quality of leaf-task execution depends on the underlying model's capabilities and available tools.
- **DAG structure is static per decomposition.** ROMA does not dynamically restructure the DAG mid-execution based on intermediate results (though a failed leaf can be re-decomposed). For tasks where the plan fundamentally changes based on early findings, interleave decomposition with execution at coarser granularity.

## Reference

**Paper:** [ROMA: Recursive Open Meta-Agent Framework for Long-Horizon Multi-Agent Systems](https://arxiv.org/abs/2602.01848v1) (Alzu'bi et al., 2026). Look for Algorithm 1 (the recursive control loop), the four-role meta-agent architecture (Section 3), dependency-aware DAG construction, and the GEPA+ prompt optimization technique for tuning executor prompts.