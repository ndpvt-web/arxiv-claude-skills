---
name: "table-as-search-formulate-long-horizon-agentic"
description: "Structured table-completion framework for long-horizon information seeking. Converts complex research queries into database tables where rows are candidates and columns are constraints/attributes, then orchestrates deep and wide search agents to fill cells systematically. Use when: 'research and compare multiple options', 'find all X that satisfy Y constraints', 'deep dive investigation across sources', 'build a comparison table from web research', 'search for entities matching criteria', 'comprehensive multi-step information gathering'."
---

# Table-as-Search: Structured Table Completion for Long-Horizon Information Seeking

This skill enables Claude to tackle complex, multi-step information-seeking tasks by reformulating them as **table completion problems**. Instead of maintaining fragile plain-text search state across many steps, Claude constructs a structured table schema where rows represent candidate entities, columns represent constraints or required attributes, and cells track search progress. Empty cells become an explicit search plan; filled cells become verified evidence. This approach -- from the Table-as-Search (TaS) framework -- unifies deep search (precise filtering), wide search (broad aggregation), and hybrid deep-wide search into a single disciplined workflow.

## When to Use

- When the user asks to **research and compare multiple entities** against a set of criteria (e.g., "Find all open-source LLM frameworks that support RLHF, list their license, GPU requirements, and community size")
- When the task requires **broad candidate discovery plus deep verification** -- the DeepWide pattern (e.g., "Find restaurants in Tokyo with Michelin stars that serve vegetarian omakase, with prices and reservation info")
- When a research query involves **multiple constraints that must all be satisfied** (e.g., "Which Python ORMs support async, have type hints, and work with PostgreSQL and SQLite?")
- When the user needs a **systematic comparison table** built from scattered sources (e.g., "Compare the top 5 vector databases on performance, pricing, and hosted options")
- When information seeking spans **many search iterations** and tracking state in prose would lose coherence
- When the user asks to "find all X that match Y" or "build me a table of Z" from external information

## Key Technique

**The core insight:** Long-horizon information seeking fails when agents track search state as unstructured text. Context windows fill with redundant results, the agent loses track of what has been found vs. what remains, and planning degrades. TaS solves this by externalizing search state into a structured table -- a database where every cell has a defined status: filled (verified), empty (pending), or N/A (not applicable).

**Schema formulation:** Every query is decomposed into a tuple `Schema = <K, C, I>` where **K** (Keys) defines candidate entities as table rows, **C** (Constraints) defines filtering requirements as columns, and **I** (Information) defines attributes to collect as additional columns. This decomposition determines the search type: if `|C| > 0` the task requires deep filtering; if `|I| > 0` it requires wide aggregation; if both, it is a DeepWide search requiring the full orchestration.

**Two-phase cell filling:** The planner alternates between *row expansion* (discovering new candidate entities via broad search to add rows) and *cell population* (filling specific attribute/constraint cells for existing rows via targeted deep search). The planner checks table saturation after each cycle -- when all required cells are filled or enough candidates pass constraints, it synthesizes the final answer from the structured evidence.

## Step-by-Step Workflow

1. **Decompose the query into a table schema.** Parse the user's request to identify: (a) the candidate entity type (what populates rows), (b) hard constraints that candidates must satisfy (constraint columns), and (c) information attributes to collect (information columns). Write this schema out explicitly.

2. **Classify the search type.** Determine whether this is a Deep Search (many constraints, few known candidates), Wide Search (many attributes to collect across many candidates), or DeepWide Search (both). This determines whether to prioritize row expansion or cell population first.

3. **Initialize the table with known candidates.** If the user provides some candidates or they are obvious, pre-populate rows. Mark all unknown cells as `PENDING`. Mark cells that clearly don't apply as `N/A`.

4. **Execute row expansion (Wide phase).** Search broadly to discover candidate entities. For each new entity found, add a row to the table with the entity name in the Key column and all other cells set to `PENDING`. Use constraint-aware search queries to prioritize likely matches.

5. **Execute cell population (Deep phase).** For each row with `PENDING` cells, perform targeted searches to fill specific attribute and constraint cells. Verify constraint cells with explicit evidence -- mark as `PASS`, `FAIL`, or the specific value found. Fill information cells with the retrieved data.

6. **Prune failing candidates.** After constraint verification, remove or flag rows where mandatory constraints are `FAIL`. This keeps the table focused and prevents wasted effort on disqualified candidates.

7. **Check saturation and iterate.** Inspect the table: are there enough qualifying rows? Are all required information cells filled for passing candidates? If not, return to step 4 (if more candidates needed) or step 5 (if more attributes needed).

8. **Synthesize the answer from the completed table.** Present the filled table as structured output. Summarize key findings, highlight the best candidates based on the user's implicit or explicit ranking criteria, and cite sources for each cell value.

9. **Present the table to the user.** Format as a clean markdown table with clear headers. Include a summary paragraph interpreting the results. Offer to drill deeper into any specific row or column.

## Concrete Examples

**Example 1: Wide Search -- Comparing Vector Databases**

```
User: "Compare the top vector databases for my RAG pipeline. I need to know
about pricing, max dimensions supported, filtering capabilities, and
whether they have a managed cloud offering."

Schema Decomposition:
  K (Keys/Rows): Vector database products
  C (Constraints): None (no hard filters)
  I (Information): pricing, max_dimensions, filtering_capabilities, managed_cloud

Search Type: Wide Search (|C|=0, |I|=4)

Step 1 - Row Expansion:
  Search: "top vector databases 2025 comparison"
  Candidates found: Pinecone, Weaviate, Qdrant, Milvus, ChromaDB, pgvector

Step 2 - Initialize Table:
  | Database  | Pricing     | Max Dimensions | Filtering      | Managed Cloud |
  |-----------|-------------|----------------|----------------|---------------|
  | Pinecone  | PENDING     | PENDING        | PENDING        | PENDING       |
  | Weaviate  | PENDING     | PENDING        | PENDING        | PENDING       |
  | Qdrant    | PENDING     | PENDING        | PENDING        | PENDING       |
  | Milvus    | PENDING     | PENDING        | PENDING        | PENDING       |
  | ChromaDB  | PENDING     | PENDING        | PENDING        | PENDING       |
  | pgvector  | PENDING     | PENDING        | PENDING        | PENDING       |

Step 3 - Cell Population (targeted searches per row):
  Search per row: "{database} pricing tiers 2025", "{database} max vector dimensions",
  "{database} metadata filtering", "{database} managed cloud service"

Final Output:
  | Database  | Pricing          | Max Dims | Filtering               | Managed Cloud |
  |-----------|------------------|----------|-------------------------|---------------|
  | Pinecone  | Free tier + usage| 20,000   | Metadata + namespace    | Yes (native)  |
  | Weaviate  | Open source/paid | 65,535   | GraphQL + hybrid        | Yes (WCD)     |
  | Qdrant    | Open source/paid | 65,535   | Payload filtering       | Yes (Cloud)   |
  | Milvus    | Open source      | 32,768   | Scalar + expression     | Yes (Zilliz)  |
  | ChromaDB  | Open source      | No limit | Metadata where clause   | No            |
  | pgvector  | Postgres license | 2,000    | Full SQL WHERE          | Via providers |
```

**Example 2: Deep Search -- Finding Packages Matching Strict Constraints**

```
User: "I need a Python logging library that supports structured JSON output,
async-safe operation, and has zero dependencies. Which options exist?"

Schema Decomposition:
  K (Keys/Rows): Python logging libraries
  C (Constraints): structured_json=YES, async_safe=YES, zero_dependencies=YES
  I (Information): None beyond constraint verification

Search Type: Deep Search (|C|=3, |I|=0)

Step 1 - Row Expansion:
  Search: "Python structured logging library JSON async"
  Candidates: structlog, loguru, python-json-logger, eliot, picologging

Step 2 - Initialize Table:
  | Library            | JSON Output | Async-Safe | Zero Deps |
  |--------------------|-------------|------------|-----------|
  | structlog          | PENDING     | PENDING    | PENDING   |
  | loguru             | PENDING     | PENDING    | PENDING   |
  | python-json-logger | PENDING     | PENDING    | PENDING   |
  | eliot              | PENDING     | PENDING    | PENDING   |
  | picologging        | PENDING     | PENDING    | PENDING   |

Step 3 - Cell Population (verify each constraint per candidate):
  Search: "{library} structured JSON logging", "{library} async safe thread safe",
  "{library} dependencies requirements.txt pypi"

Step 4 - Pruned Table:
  | Library            | JSON Output | Async-Safe | Zero Deps | Status |
  |--------------------|-------------|------------|-----------|--------|
  | structlog          | PASS        | PASS       | FAIL (1)  | FAIL   |
  | loguru             | PASS        | PASS       | FAIL (1)  | FAIL   |
  | python-json-logger | PASS        | PASS       | FAIL (1)  | FAIL   |
  | eliot              | PASS        | FAIL       | FAIL      | FAIL   |
  | picologging        | FAIL        | PASS       | PASS      | FAIL   |

Result: "No library satisfies all three constraints simultaneously. structlog
and loguru come closest -- both support JSON and async but each carry one
dependency. Consider relaxing the zero-dependencies constraint."
```

**Example 3: DeepWide Search -- Market Research with Constraints and Attributes**

```
User: "Find CI/CD platforms that support monorepo builds natively, have a
free tier for open source, and list their build minutes, parallelism
limits, and self-hosted runner support."

Schema Decomposition:
  K (Keys/Rows): CI/CD platforms
  C (Constraints): monorepo_native=YES, free_oss_tier=YES
  I (Information): build_minutes, parallelism_limit, self_hosted_runners

Search Type: DeepWide (|C|=2, |I|=3)

Phase 1 - Row Expansion + Constraint Check:
  Search: "CI/CD platforms monorepo support free open source"
  Candidates: GitHub Actions, GitLab CI, CircleCI, BuildKite, Nx Cloud,
              Semaphore, Earthly

Phase 2 - Constraint Verification (Deep):
  Verify monorepo support and free OSS tier for each candidate.
  Prune: Earthly (no native CI), BuildKite (no free OSS tier)

Phase 3 - Information Collection (Wide):
  For passing candidates, fill build_minutes, parallelism, self-hosted columns.

Final Output:
  | Platform       | Monorepo | Free OSS | Build Min/mo | Parallelism | Self-Hosted |
  |----------------|----------|----------|--------------|-------------|-------------|
  | GitHub Actions | PASS     | PASS     | Unlimited*   | 20 parallel | Yes         |
  | GitLab CI      | PASS     | PASS     | 400 min      | Varies      | Yes         |
  | CircleCI       | PASS     | PASS     | 400K credits | 30x         | Yes         |
  | Nx Cloud       | PASS     | PASS     | 500 hrs      | Unlimited   | Yes         |
  | Semaphore      | PASS     | PASS     | 1,300 min    | 4 parallel  | Yes (paid)  |

  * Subject to concurrency limits. All data verified as of search date.
```

## Best Practices

- **Do:** Always write out the schema decomposition (`K`, `C`, `I`) explicitly before searching. This prevents scope drift during long investigations.
- **Do:** Use `PENDING`, `PASS`, `FAIL`, `N/A` as cell states consistently. This makes search progress unambiguous at a glance.
- **Do:** Prune failing candidates early. Once a hard constraint is `FAIL`, stop filling information cells for that row -- it saves significant effort.
- **Do:** Separate row expansion from cell population into distinct phases. Mixing them leads to the same incoherence TaS is designed to prevent.
- **Avoid:** Stuffing full search result text into table cells. Cells should contain concise, verified values. Keep raw evidence in notes or citations, not in the table itself.
- **Avoid:** Expanding the table schema mid-search unless the user explicitly changes requirements. Adding columns or redefining constraints after starting causes cascading re-work.

## Error Handling

- **No candidates found during row expansion:** Broaden search terms, remove optional constraints temporarily, or try alternative phrasings. If still empty, report to the user that the search space may be too narrow.
- **Conflicting information for a cell:** Record both values with sources and mark the cell as `DISPUTED`. Present both options to the user for resolution rather than silently picking one.
- **Search saturation without full coverage:** If repeated searches yield no new rows or cell values, stop and present the partial table with a clear indication of which cells remain `PENDING` and why.
- **Too many candidates (row explosion):** Cap rows at a reasonable number (10-15 for comparisons, 20-30 for exhaustive lists). Prioritize by relevance or apply additional constraints to narrow the field.
- **Constraint ambiguity:** If a constraint is subjective (e.g., "good documentation"), ask the user to define a measurable proxy before starting the search, or define one explicitly and state the assumption.

## Limitations

- This approach works best for **entity-centric queries** where candidates can be enumerated as rows. It is less suitable for abstract reasoning tasks, opinion synthesis, or narrative research where there is no natural tabular structure.
- The quality of results depends entirely on **search tool availability**. Without web search access, the framework can only organize information already in context -- it cannot discover new candidates.
- **Highly dynamic information** (real-time prices, live availability) may be stale by the time the table is completed. Note the search date and warn users about time-sensitive cells.
- For queries with **hundreds of potential candidates**, the table becomes unwieldy. In such cases, apply stricter initial constraints or use sampling rather than exhaustive enumeration.
- The framework assumes candidates are **relatively independent**. Queries where entities have complex interdependencies (e.g., dependency graphs, causal chains) may need a different structural representation.

## Reference

**Paper:** [Table-as-Search: Formulate Long-Horizon Agentic Information Seeking as Table Completion](https://arxiv.org/abs/2602.06724v1) (Lan et al., 2026). Look for: the `Schema = <K, C, I>` formalization, the row-expansion vs. cell-population orchestration loop, and the experimental results showing 40%+ improvement on hard DeepWide tasks over ReAct baselines.

**Code:** [github.com/AIDC-AI/Marco-Search-Agent](https://github.com/AIDC-AI/Marco-Search-Agent) -- reference implementation with Wide/Deep sub-agents and MongoDB-backed table state management.