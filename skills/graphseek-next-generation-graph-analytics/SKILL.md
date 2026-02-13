---
name: "graphseek-next-generation-graph-analytics"
description: "Build LLM-powered graph analytics systems using the GraphSeek two-plane architecture: a Semantic Catalog for planning over graph schemas and operations, separated from deterministic database-grade query execution. Use when: 'query a property graph with natural language', 'build a graph analytics pipeline', 'connect LLM reasoning to Neo4j/Cypher', 'analyze a knowledge graph without writing queries', 'create a semantic layer for graph databases', 'multi-step graph traversal with LLM planning'."
---

# GraphSeek: Next-Generation Graph Analytics with LLMs

This skill enables Claude to design and implement LLM-powered graph analytics systems based on the GraphSeek architecture. The core technique replaces brittle direct translation of natural language to graph queries (e.g., Cypher, Gremlin) with a two-plane design: a **Semantic Plane** where the LLM plans over a compact **Semantic Catalog** describing the graph schema and available operations, and an **Execution Plane** where deterministic code compiles those plans into real database queries and runs them at full scale. This separation yields 86%+ success rates on complex property graph analytics while keeping token costs stable and low.

## When to Use

- When the user wants to query a graph database (Neo4j, Kùzu, etc.) using natural language instead of writing Cypher/Gremlin directly
- When building a multi-step graph analytics pipeline that chains traversals, aggregations, and filters
- When designing an LLM agent layer on top of an existing property graph with heterogeneous node/edge types
- When the user asks to "make my graph database accessible to non-technical users"
- When implementing a semantic layer or middleware between an LLM and a graph backend
- When the user has complex graph queries that require planning (path finding, multi-hop traversals, temporal slicing) rather than single-shot generation
- When an existing LangChain-style graph QA chain is too brittle or token-expensive for production use

## Key Technique

**The Semantic Catalog abstraction.** Instead of prompting an LLM with the full graph schema and asking it to generate a Cypher query in one shot, GraphSeek introduces a Semantic Catalog -- a compact (~2,000 token) document containing two families of descriptors: (1) **Schema Descriptors** that annotate each node/edge type with a short natural-language explanation of its domain meaning, and (2) **Operator Descriptors** that define intermediate graph operations (k-hop traversal, neighborhood aggregation, temporal slicing, entity lookup) with typed signatures and behavioral annotations. The LLM plans by selecting and parameterizing operators from this catalog rather than synthesizing raw query strings.

**Two-plane execution model.** The Semantic Plane (LLM) handles intent interpretation, operator selection, and answer generation. The Execution Plane (deterministic code) compiles operator invocations into backend-specific queries, executes them against the full dataset, and returns only O(1)-sized summaries back to the LLM context. Full results are persisted in an out-of-context artifact store and referenced by handles in subsequent steps. This prevents context blowup and enables multi-step refinement: if a query returns empty results (e.g., due to floating-point precision), the system re-plans with a corrected operator rather than stuffing more tokens into the prompt.

**Why this beats direct generation.** Direct NL-to-Cypher generation fails on property graphs because schemas are large and heterogeneous, domain terms are ambiguous (e.g., "baseline" means `assemblyTier=0` in manufacturing), and multi-step analytics require chaining multiple queries with intermediate reasoning. By routing through the catalog, the LLM never sees raw schema DDL -- it sees curated semantic annotations that encode domain knowledge the model cannot infer. This achieves 86% success rates where enhanced LangChain approaches fail, with 2-3 second median latency versus 40-60x slower alternatives.

## Step-by-Step Workflow

1. **Model the graph schema as Schema Descriptors.** For each node label and relationship type in the property graph, write a 1-2 sentence natural-language annotation explaining its domain meaning, key properties, and how it connects to other types. Store these in a structured format (JSON or YAML). Example: `DriveAssembly: "Represents a partially assembled propulsion subsystem within production. Key properties: assemblyTier (int), factoryCode (string), unitCost (float)."`

2. **Define the Operator Catalog.** Create a library of typed, schema-validated graph operators. Each operator has: a name, a natural-language description of its behavior, a JSON Schema for its parameters, and a compilation function that emits the backend query. Start with these four categories:
   - **Data operators**: entity lookup, k-hop traversal, filtered aggregation, temporal slice
   - **Retrieval operators**: load prior results by handle, convert graph results to tabular format
   - **Presentation operators**: format as table, generate chart data, produce subgraph summary
   - **Agent utilities**: spawn isolated subtask, controlled fallback on failure

3. **Assemble the Semantic Catalog.** Combine schema descriptors and operator descriptors into a single document under ~2,000 tokens. This is the only graph-related context the LLM receives. Validate that every node/edge type is covered and every operator references valid schema elements.

4. **Implement the Execution Plane.** Write deterministic functions that: (a) accept a typed operator invocation `(operator_name, parameters)`, (b) compile it to a backend query (Cypher for Neo4j, SQL for relational, etc.), (c) execute it against the database, (d) store full results in an artifact store keyed by a handle ID, and (e) return a compact summary (row count, top-N sample, schema of result) to the caller.

5. **Build the Semantic Plane agent loop.** Implement a 4-stage loop per step:
   - **Synthesis**: Given the user's NL request and the Semantic Catalog, select an operator and fill its parameters.
   - **Execution**: Pass the typed invocation to the Execution Plane; receive the compact summary.
   - **Generation**: Produce a candidate answer or intermediate reasoning from the summary.
   - **Decision**: Determine whether to halt (answer is complete) or continue (need another operator call).

6. **Wire up the artifact store.** Implement a key-value store (in-memory dict, Redis, or filesystem) where each execution result is saved under a unique handle. The LLM context only ever contains handles and summaries, never raw result sets. Subsequent operator calls can reference prior handles as inputs.

7. **Add self-correction logic.** When an operator execution returns zero results or an error, do NOT re-prompt with more context. Instead, have the LLM re-plan: analyze the compact error summary, select a different operator or adjust parameters (e.g., switch from equality filter to range filter), and re-execute. Cap retries at 3.

8. **Support parallel operator invocation.** For queries like "compare metrics across three entities," emit multiple operator calls concurrently rather than sequentially. Collect all results into the artifact store before the generation step.

9. **Validate with dry runs.** Before deploying a new operator or schema change, dry-run the compiled query against the live schema (without executing) to catch type mismatches, missing properties, or invalid traversals.

10. **Expose a clean API.** Wrap the system in an endpoint that accepts NL queries and returns structured answers (JSON with the answer, confidence, execution trace, and artifact handles for drill-down).

## Concrete Examples

**Example 1: Manufacturing Graph -- Aggregation Query**

User: "How many baseline drive assemblies does each factory produce?"

Approach:
1. The Semantic Plane receives the NL query plus the Semantic Catalog.
2. It resolves "baseline" to the schema descriptor for DriveAssembly where `assemblyTier = 0`.
3. It selects the `filtered_aggregation` operator: `{operator: "filtered_aggregation", params: {node_type: "DriveAssembly", filter: {assemblyTier: 0}, group_by: "factoryCode", aggregate: "count"}}`.
4. The Execution Plane compiles this to Cypher:
   ```cypher
   MATCH (d:DriveAssembly)
   WHERE d.assemblyTier = 0
   RETURN d.factoryCode AS factory, count(d) AS total
   ORDER BY total DESC
   ```
5. Full results are stored as artifact `handle_001`. Summary returned: `{rows: 4, columns: ["factory", "total"], sample: [["F-East", 142], ["F-West", 98]]}`.
6. The LLM generates the final answer from the summary.

Output:
```json
{
  "answer": "Factory F-East produces 142 baseline drive assemblies, F-West produces 98, F-North produces 76, and F-South produces 61.",
  "artifact": "handle_001",
  "operators_used": ["filtered_aggregation"],
  "steps": 1
}
```

**Example 2: Knowledge Graph -- Multi-Hop Path Traversal**

User: "What is the unique production plan for vehicle model PX-206?"

Approach:
1. The LLM resolves "PX-206" to node type `VehicleModel` via schema descriptors.
2. It selects `entity_lookup`: `{operator: "entity_lookup", params: {node_type: "VehicleModel", filter: {modelCode: "PX-206"}}}`. Execution returns handle `h1` with the node.
3. It then selects `k_hop_traversal`: `{operator: "k_hop_traversal", params: {start_handle: "h1", relationships: ["CONSUMED_IN", "GENERATES", "PROCESSED_AT"], max_hops: 3}}`. Execution returns handle `h2` with the subgraph.
4. It selects `subgraph_summary`: `{operator: "subgraph_summary", params: {handle: "h2"}}` to get a readable summary.
5. The LLM synthesizes the production plan narrative from the summary.

Output:
```
The production plan for PX-206 involves 3 stages:
1. Module assembly at Plant-A (DriveAssembly tier 0-2)
2. Integration via CONSUMED_IN at Line-7
3. Final processing at QA-Station-12 via PROCESSED_AT

Subgraph: 14 nodes, 19 relationships across 4 node types.
Artifact handle: h2 (available for further drill-down)
```

**Example 3: Self-Correction on Failed Query**

User: "Which module has the highest unit cost?"

Approach:
1. The LLM selects `filtered_aggregation` with `aggregate: "max"` on `unitCost`. Execution returns handle `h1`.
2. It then selects `entity_lookup` with `filter: {unitCost: 847.32}` (exact max value). Execution returns 0 rows due to floating-point representation.
3. Self-correction triggers: the LLM recognizes the empty result, re-plans with a ranking operator: `{operator: "top_k", params: {node_type: "Module", sort_by: "unitCost", order: "desc", k: 1}}`.
4. Execution returns the correct module. Total steps: 3 (including 1 retry).

## Best Practices

- **Do** write schema descriptors that encode domain jargon the LLM cannot infer. "Baseline" meaning `assemblyTier=0` is exactly the kind of knowledge that belongs in the catalog, not in the prompt.
- **Do** keep the Semantic Catalog under 2,000 tokens. If it grows beyond that, split into domain-specific sub-catalogs and select the relevant one based on the query topic.
- **Do** return only compact summaries (row counts, top-N samples, schema) to the LLM context. Never pass full result sets through the LLM -- store them in the artifact store.
- **Do** validate operator parameters against JSON Schema before compiling to a backend query. Catch type errors at the plan level, not at the database level.
- **Avoid** generating raw Cypher/Gremlin/SPARQL directly from natural language. This is the single biggest failure mode in graph QA systems -- the catalog-mediated approach exists specifically to prevent it.
- **Avoid** stuffing more context into the prompt when a query fails. The correct response is to re-plan with a different operator or adjusted parameters, not to add schema DDL or example queries.
- **Avoid** exposing the full graph schema to the LLM. The schema descriptors are a curated projection -- include only what's needed for the query domain.

## Error Handling

| Failure Mode | Detection | Recovery |
|---|---|---|
| Operator returns empty results | Summary shows `rows: 0` | Re-plan: switch filter strategy (equality to range, exact to top-k) |
| Schema mismatch (property not found) | Compilation error before execution | Validate against live schema; suggest closest matching property from catalog |
| Timeout on large traversal | Execution plane timeout | Add `LIMIT` clause or reduce `max_hops`; return partial results with warning |
| Ambiguous NL term | Multiple schema descriptors match | Ask the user to disambiguate, listing the matching domain concepts |
| Context overflow | Token count exceeds model limit | Prune execution trace history; keep only latest 2-3 step summaries plus artifact handles |
| Cyclic multi-step loop | Step count exceeds cap (default: 10) | Force halt, return best intermediate answer with explanation of what blocked completion |

## Limitations

- **Schema descriptor quality is critical.** The system is only as good as the annotations in the Semantic Catalog. Poorly written or incomplete descriptors will cause the LLM to select wrong operators or misinterpret domain terms. Expect to iterate on descriptors.
- **Not a replacement for expert query optimization.** The Execution Plane compiles operators to straightforward queries. Complex performance tuning (index hints, query plan optimization) still requires a database expert.
- **Single-database scope per catalog.** Each Semantic Catalog targets one graph database instance. Cross-database federation requires multiple catalogs and an orchestration layer.
- **Operator library coverage.** The system cannot answer queries that require operations not in the catalog. Novel analytics (e.g., custom graph algorithms) need new operator implementations, not just prompt engineering.
- **LLM reasoning ceiling.** For queries requiring deep multi-step mathematical reasoning over graph structures (e.g., "find the minimum vertex cover"), the LLM planning layer may fail even with correct operators available.

## Reference

[GraphSeek: Next-Generation Graph Analytics with LLMs](https://arxiv.org/abs/2602.11052v1) -- Besta et al., 2026. Focus on Section 3 (Semantic Catalog abstraction), Section 4 (two-plane architecture and the syn/exec/gen/dec loop), and Section 6 (experimental results showing 86% success rates and token efficiency gains over LangChain baselines).