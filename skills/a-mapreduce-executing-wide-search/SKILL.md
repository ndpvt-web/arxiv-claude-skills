---
name: "a-mapreduce-executing-wide-search"
description: "Execute large-scale breadth-oriented search and retrieval tasks using the A-MapReduce pattern: decompose a wide query into a task matrix, dispatch parallel map agents for independent retrieval, then reduce partial results into a unified structured table. Triggers: 'find all X that match Y across a large set', 'build a comparison table of N items', 'search for every instance of X', 'collect attributes for a list of entities', 'wide search across many sources', 'gather structured data on hundreds of items'."
---

# A-MapReduce: Executing Wide Search via Agentic MapReduce

This skill enables Claude to tackle **wide search** problems -- tasks requiring breadth-oriented retrieval across many entities or sources rather than deep iterative reasoning on a single thread. Based on the A-MapReduce framework (Chen et al., 2026), it recasts wide retrieval as a horizontally structured problem: decompose the query into a task matrix of entities and attributes, dispatch parallel map agents to retrieve missing values independently, then reduce partial tables into a single validated result. This approach achieves 5-17% F1 improvements over sequential baselines while cutting execution time by ~46%.

## When to Use

- When the user asks to **build a structured comparison table** across many entities (e.g., "Compare the pricing, features, and support tiers of the top 20 CI/CD platforms")
- When the user needs to **collect the same set of attributes** for a large list of items (e.g., "For each of these 50 npm packages, find the license, last release date, and weekly downloads")
- When the task involves **broad entity discovery** followed by attribute grounding (e.g., "Find all YC W24 companies in the healthcare space and list their funding, team size, and product")
- When a search query naturally decomposes into **independent, parallelizable subtasks** with no cross-dependencies between rows
- When the user asks to **audit, survey, or inventory** a category (e.g., "List every Rust HTTP framework with its async runtime, GitHub stars, and last commit date")
- When sequential search would be impractically slow due to the **number of targets** (dozens to hundreds of entities)

## Key Technique

### Wide Search vs. Deep Search

Most agentic search systems are optimized for **deep search**: iterative, vertically structured reasoning where each step builds on the previous one (e.g., multi-hop question answering). **Wide search** is fundamentally different -- it requires covering a large horizontal surface of entities and attributes with minimal cross-dependencies between retrieval units. Sequential deep-search agents get stuck in expansive objectives and suffer from long-horizon execution drift.

### The A-MapReduce Paradigm

A-MapReduce borrows from the classical MapReduce programming model but adapts it for agentic retrieval. The core abstraction is a **decision tuple** `Theta_q = (M_q, P_q, B_q)`:

- **Task Matrix (`M_q`)**: An N-by-K table where N is the number of target entities and K is the number of known + required attributes. Each row represents one entity; columns hold known values and empty cells to be filled.
- **Template (`P_q`)**: A query-specific string with placeholders aligned to matrix columns, used to instantiate atomic search tasks. For entity `i`, the atomic task is `t_i = fill(P_q, M_q[i, :])`.
- **Batching Strategy (`B_q`)**: How atomic tasks are grouped for parallel agents -- `per_atom` (one entity per batch, best for diverse tasks), `by_attr` (group by shared attribute, good for homogeneous lookups), or `open` (agent-crafted batches balancing context reuse and parallelism).

### Experiential Memory for Progressive Improvement

The system maintains a memory store of past executions: `(query, decision, trace, utility)`. When a new query arrives, it retrieves semantically similar high-utility and low-utility records via contrastive retrieval, composing a **decision anchor** that biases the decomposition toward strategies that worked on similar tasks. Over repeated runs, hints are distilled from execution clusters into reusable guidance, enabling continual improvement without retraining.

## Step-by-Step Workflow

1. **Classify the task as wide search.** Confirm the query requires retrieving the same type of information across many entities (breadth) rather than deeply exploring a single entity (depth). If the task is fewer than ~5 entities with complex interdependencies, use standard sequential search instead.

2. **Define the output schema `S`.** Enumerate the exact columns the final table must contain. Specify data types and constraints (e.g., "funding: dollar amount or 'Undisclosed'", "license: SPDX identifier"). This schema drives both decomposition and validation.

3. **Construct the task matrix `M_q`.** List all N target entities as rows. Populate any columns where values are already known from the user's input. Leave cells blank where retrieval is needed. If entities themselves must be discovered first, run a preliminary discovery phase to populate the entity list before proceeding.

4. **Design the query template `P_q`.** Write a fill-in-the-blank search query that, when instantiated with a row's known values, produces an effective atomic search task. Example: `"What is the {attribute} of {company_name}, a {sector} company founded in {year}?"`. Test the template mentally against 2-3 rows to verify it produces sensible queries.

5. **Select the batching strategy `B_q`.** Choose `per_atom` when entities are diverse and independent (default). Choose `by_attr` when entities share a grouping dimension that aids retrieval (e.g., all companies in the same industry). Choose `open` when the optimal grouping is unclear and agents should decide.

6. **Dispatch map agents in parallel.** Partition the task matrix into batches according to `B_q`. For each batch, spawn an independent agent (using the Task tool with `subagent_type="general-purpose"`) that receives its subset of rows, the template, the schema, and instructions to fill missing cells via search, web fetch, or codebase exploration. Each agent returns a partial table `Y_k`.

7. **Reduce partial tables via union and validation.** Merge all `Y_k` into a single table `Y = union(Y_1, ..., Y_m)`. Validate against schema `S`: check for missing cells, type mismatches, and contradictions. Flag rows where agents returned conflicting values.

8. **Run delta-patch repair rounds.** For any incomplete or conflicting cells, construct targeted repair queries and dispatch a small number of focused agents to resolve them. This is a lightweight second MapReduce pass on only the gaps, not a full re-execution.

9. **Format and deliver the final result.** Present the validated table in the format most useful to the user (Markdown table, JSON, CSV). Include a completeness summary (e.g., "48/50 entities fully populated, 2 entities missing funding data").

10. **Store execution hints for reuse.** If the session involves repeated similar queries, record which decomposition strategy and template worked well, so subsequent runs can start from a better decision anchor.

## Concrete Examples

**Example 1: Comparing npm packages**

```
User: "I have a list of 30 npm packages for form validation. For each one,
find the weekly downloads, bundle size, TypeScript support, last publish
date, and GitHub stars."

Approach:
1. Schema: [package_name, weekly_downloads, bundle_size_kb, typescript_support,
   last_publish, github_stars]
2. Task matrix: 30 rows (one per package), package_name column pre-filled
3. Template: "npm package {package_name}: weekly downloads, bundle size,
   TypeScript support status, last publish date, GitHub stars"
4. Batching: per_atom (packages are independent)
5. Dispatch: 6 parallel agents, each handling 5 packages
   - Agent 1 queries npm registry API + GitHub API for packages 1-5
   - Agent 2 handles packages 6-10
   - ... (all run concurrently)
6. Reduce: Merge 6 partial tables, validate types (downloads = number,
   bundle_size = number, etc.)
7. Delta-patch: 2 packages had no bundlephobia data -- dispatch one
   repair agent to check alternatives

Output:
| Package      | Downloads/wk | Bundle (kB) | TS  | Last Publish | Stars |
|-------------|-------------|-------------|-----|-------------|-------|
| zod         | 12.4M       | 13.4        | Yes | 2026-01-15  | 35.2k |
| yup         | 5.1M        | 22.1        | Yes | 2025-11-02  | 22.8k |
| joi         | 8.9M        | 45.3        | No  | 2025-06-20  | 20.9k |
| ...         | ...         | ...         | ... | ...         | ...   |

Completeness: 30/30 entities fully populated.
```

**Example 2: Codebase-wide API audit**

```
User: "Audit every REST endpoint in our Express app. For each one, find
the HTTP method, route path, authentication requirement, rate limit
config, and whether it has integration tests."

Approach:
1. Schema: [method, route, auth_required, rate_limit, has_tests]
2. Discovery phase: Grep for router.get/post/put/delete/patch across
   src/routes/ to build entity list -- finds 47 endpoints
3. Task matrix: 47 rows, method + route pre-filled from grep
4. Template: "For endpoint {method} {route}: check middleware chain for
   auth guards, check rate-limit config, search test files for coverage"
5. Batching: by_attr grouped by route file (endpoints in the same file
   share middleware context)
6. Dispatch: 8 parallel agents, each handling one route file's endpoints
   - Agent reads the route file, traces middleware, checks test coverage
7. Reduce: Merge results, flag any endpoint where auth status is ambiguous
8. Delta-patch: 3 endpoints use inherited middleware -- dispatch agent to
   trace the app-level middleware chain

Output:
| Method | Route              | Auth     | Rate Limit  | Tests |
|--------|--------------------|----------|-------------|-------|
| GET    | /api/users         | JWT      | 100/min     | Yes   |
| POST   | /api/users         | JWT+Admin| 20/min      | Yes   |
| GET    | /api/health        | None     | None        | No    |
| DELETE | /api/users/:id     | JWT+Admin| 10/min      | Yes   |
| ...    | ...                | ...      | ...         | ...   |

Completeness: 44/47 fully resolved. 3 endpoints flagged for manual review
(dynamic middleware assignment).
```

**Example 3: Research survey across repositories**

```
User: "Find all open-source vector databases. For each, list the
language, indexing algorithms supported, max tested scale, license,
and cloud offering."

Approach:
1. Discovery phase first: search for "vector database" across GitHub,
   awesome-lists, and comparison articles to build entity list
2. Schema: [name, language, index_algorithms, max_scale, license, cloud]
3. Task matrix: 18 discovered databases, name column pre-filled
4. Template: "{name} vector database: primary language, supported index
   algorithms (HNSW, IVF, etc.), maximum tested dataset scale, license
   type, managed cloud offering availability"
5. Batching: per_atom (each DB is independent)
6. Dispatch: 6 agents, 3 databases each
   - Each agent checks the project's GitHub README, docs, and benchmarks
7. Reduce: Merge, normalize license names to SPDX, validate algorithm
   names against known set
8. Delta-patch: 2 newer projects lacked benchmark data -- repair agent
   checks their docs and blog posts

Output: Structured table with 18 rows, completeness notes per cell.
```

## Best Practices

**Do:**
- Define the output schema explicitly before decomposition. Ambiguous schemas lead to incompatible partial results that are hard to merge.
- Pre-fill as many columns as possible in the task matrix. Known values dramatically improve search query quality and reduce false matches.
- Use `per_atom` batching as the default. It maximizes parallelism and avoids cascading failures where one bad batch blocks others.
- Include type constraints in the schema (numbers, dates, enums) so the reduce phase can catch obviously wrong values automatically.
- Run a lightweight discovery phase when the entity list itself is unknown, then proceed to the full MapReduce for attribute retrieval.

**Avoid:**
- Forcing wide search on tasks with deep interdependencies. If retrieving entity B's attributes depends on entity A's results, this is a deep search problem -- use sequential reasoning instead.
- Creating batches larger than ~10 entities. Overly large batches degrade individual agent performance and negate the parallelism benefit.
- Skipping the delta-patch phase. First-pass retrieval almost never achieves 100% coverage; the repair round is essential for completeness.
- Using the `open` batching strategy without good reason. It adds agent decision overhead; prefer deterministic batching unless entity grouping is genuinely unclear.

## Error Handling

| Problem | Detection | Resolution |
|---------|-----------|------------|
| Agent returns empty results for a batch | Reduce phase finds rows with all cells blank | Re-dispatch that batch with a refined template or alternative search strategy |
| Conflicting values across agents | Two agents report different values for the same cell | Flag for delta-patch; dispatch a tiebreaker agent with both values as context |
| Entity not found | Search returns no relevant results | Mark row as "Not Found" with confidence note; do not fabricate data |
| Schema mismatch | Reduce validation catches wrong types | Return row to repair queue with explicit type correction instructions |
| Rate limiting or API failures | Agent reports tool errors | Retry with exponential backoff; redistribute failed batch to other agents |
| Task matrix too large (>200 entities) | Initial size check | Split into multiple MapReduce passes of ~50-100 entities each |

## Limitations

- **Not suitable for deep reasoning tasks.** If answering requires multi-hop inference where each step depends on the previous, standard sequential agentic search is more appropriate.
- **Entity list must be enumerable.** The task matrix requires knowing (or discovering) the set of target entities upfront. Open-ended queries like "find something interesting" don't decompose well.
- **Diminishing returns on very small tasks.** For fewer than ~5 entities, the overhead of decomposition, dispatch, and reduce exceeds the cost of sequential execution.
- **Experiential memory requires repeated usage.** The progressive improvement from memory only kicks in after multiple similar queries. One-off tasks don't benefit from the memory system.
- **Result quality depends on atomic task quality.** If the query template produces poor search queries, parallelizing bad queries just produces bad results faster. Template design is critical.

## Reference

**Paper:** Chen, M., Zhang, G., Chang, H., Guo, Y., & Zhou, S. (2026). *A-MapReduce: Executing Wide Search via Agentic MapReduce.* arXiv:2602.01331v1. [https://arxiv.org/abs/2602.01331v1](https://arxiv.org/abs/2602.01331v1)

**Key insight:** Wide search is fundamentally a horizontal parallelization problem, not a sequential reasoning problem. The decision tuple `(TaskMatrix, Template, BatchStrategy)` provides a minimal but complete specification for decomposing any wide retrieval task into independent atomic units that can be executed by parallel agents and merged via structured aggregation.

**Code:** [https://github.com/mingju-c/AMapReduce](https://github.com/mingju-c/AMapReduce)