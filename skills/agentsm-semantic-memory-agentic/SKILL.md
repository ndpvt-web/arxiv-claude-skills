---
name: "agentsm-semantic-memory-agentic"
description: "Agentic Text-to-SQL with semantic memory that captures and reuses structured execution traces. Use when: 'write SQL for this database', 'query this schema', 'generate SQL from natural language', 'text to SQL with complex schema', 'help me explore this database and answer questions', 'build a reusable SQL generation pipeline'."
---

# AgentSM: Semantic Memory for Agentic Text-to-SQL

This skill enables Claude to convert natural language questions into SQL by building and leveraging **semantic memory** -- structured, reusable records of prior database exploration and query execution. Rather than treating each question independently, Claude captures execution traces as interpretable structured programs (markdown with semantic headers), stores them per-database, and retrieves relevant traces to guide future reasoning. This eliminates redundant schema exploration, reduces token usage by ~25%, and produces more accurate SQL by systematically reusing proven reasoning paths. The technique is drawn from AgentSM (Biswal et al., 2026), which achieved state-of-the-art 44.8% on Spider 2.0 Lite.

## When to Use

- When the user asks to write SQL queries against a database they've described or connected to
- When working with large, complex schemas (many tables, nested structures, multiple joins) where brute-force exploration wastes context
- When the user has multiple natural language questions against the same database -- semantic memory compounds in value
- When generating SQL for enterprise databases with diverse dialects (Snowflake, BigQuery, PostgreSQL, etc.)
- When the user needs to debug or optimize a Text-to-SQL pipeline that produces inconsistent results
- When building an automated question-answering system over structured data
- When the user says things like "query this database," "write SQL for," "translate this question to SQL," or "explore this schema"

## Key Technique

**Structured Semantic Memory over Raw Traces.** Traditional agentic Text-to-SQL systems either discard execution history or store it as raw scratchpads that degrade retrieval quality ("lost-in-the-middle" effect). AgentSM instead parses each execution trajectory into structured sections with descriptive headers -- separating *data exploration* (schema inspection, column sampling), *query execution* (CTE construction, joins, filtering), and *validation* (result checking, error correction). Each section gets an LLM-generated natural language summary. This structured format achieves ~2x the accuracy of unstructured traces in retrieval experiments.

**Database-Scoped Retrieval with Semantic Similarity.** When a new question arrives, the system filters the memory store to traces from the same database, then ranks by semantic similarity between the new question and stored questions. This works because semantically similar questions tend to probe the same tables and require similar exploration paths. The retrieved trace's exploration phase is injected into the agent's context *before* it begins work, allowing it to skip redundant schema discovery and jump directly to query construction.

**Synthetic Memory Bootstrapping.** For databases with no prior query history, AgentSM synthesizes curated questions using the schema and any available documentation, then executes them to populate the memory store. Each database gets at least one synthetic question, with additional questions allocated proportionally to schema complexity. This ensures the memory is useful from the first real query.

## Step-by-Step Workflow

1. **Catalog the target database schema.** Collect all table names, column names, data types, primary/foreign keys, and any documentation or external knowledge files. For large schemas, use vector search (e.g., embeddings with FAISS) to index schema elements for later retrieval.

2. **Check semantic memory for relevant prior traces.** Filter stored traces to those from the same database. Rank by semantic similarity between the new natural language question and stored questions. If a high-similarity match exists (same tables likely needed), retrieve its structured trace.

3. **Inject the exploration phase from the retrieved trace.** Load the "data exploration" sections of the matched trace into context. This gives the agent pre-built knowledge of relevant tables, column meanings, data distributions, and edge cases -- without re-executing exploratory queries.

4. **Perform schema linking.** Using a dedicated schema-linking pass (with vector search over schema embeddings), identify the specific tables and columns relevant to the current question. Budget this step tightly (aim for ~5 tool calls maximum) to avoid runaway exploration. Cross-reference with the retrieved trace to validate table selections.

5. **Generate the SQL query.** With the linked schema and exploration context in hand, construct the SQL. Use CTEs for complex multi-step logic. Match the target SQL dialect (Snowflake, PostgreSQL, BigQuery, etc.). The agent should integrate SQL generation directly rather than delegating to a sub-agent, to avoid context loss across handoffs.

6. **Execute and validate the query.** Run the generated SQL against the database. Check that results are non-empty, column types match expectations, and aggregation logic is sound. If execution fails, diagnose the error (syntax, missing table, type mismatch) and retry with corrections.

7. **Classify and store the execution trace.** Parse the full trajectory into phases using pattern matching: file/schema operations map to "exploration," CTE/JOIN/WHERE clauses map to "execution," result inspection maps to "validation." Assign each section a descriptive header and natural language summary.

8. **Build composite tools from recurring patterns.** If certain tool sequences repeat frequently across traces for this database (e.g., "get file extension then get DDL" always co-occur), merge them into a single composite tool. Only combine tools within the same reasoning phase to preserve semantic coherence.

9. **Synthesize coverage questions for new databases.** When encountering a database with no stored traces, generate 1+ synthetic natural language questions that exercise the schema's key tables and relationships. Execute these to populate the memory store before handling real user queries.

10. **Return the final answer with provenance.** Present the SQL query, execution results, and a brief explanation of which memory traces (if any) informed the reasoning. This transparency helps users trust and debug the output.

## Concrete Examples

**Example 1: Repeated queries against the same e-commerce database**

```
User: "How many orders were placed last month by customers in California?"
(Database: e-commerce with 47 tables, Snowflake dialect)

Approach:
1. Check semantic memory -- find a prior trace for "total revenue by state
   last quarter" on this same database.
2. Load the exploration phase from that trace. It already documents:
   - orders table uses `created_at` (TIMESTAMP_NTZ), not `order_date`
   - customer state is in `customers.billing_state`, not `customers.state`
   - orders join to customers via `orders.customer_id = customers.id`
3. Skip schema exploration entirely. Go straight to query construction:

   SELECT COUNT(DISTINCT o.id) AS order_count
   FROM orders o
   JOIN customers c ON o.customer_id = c.id
   WHERE c.billing_state = 'CA'
     AND o.created_at >= DATE_TRUNC('month', CURRENT_DATE - INTERVAL '1 month')
     AND o.created_at < DATE_TRUNC('month', CURRENT_DATE);

4. Execute, validate (non-zero result, reasonable magnitude), store trace.

Output:
| order_count |
|-------------|
| 3,847       |

Memory reuse saved ~8 exploration steps and ~40% of token budget.
```

**Example 2: First query on an unfamiliar database**

```
User: "What's the average response time for critical tickets this year?"
(Database: IT service management, PostgreSQL, no prior traces)

Approach:
1. No semantic memory exists for this database. Trigger synthetic bootstrapping:
   - Read schema: find tables `tickets`, `ticket_priorities`, `ticket_events`,
     `sla_policies`, `agents`, `departments` (23 tables total).
   - Generate synthetic question: "List all ticket categories with their
     average resolution time."
   - Execute synthetic question to populate memory with exploration of
     ticket lifecycle tables.

2. Now handle the real question. Retrieve the synthetic trace's exploration:
   - `tickets.priority_id` links to `ticket_priorities.id`
   - `ticket_priorities.name` contains 'Critical', 'High', 'Medium', 'Low'
   - Response time = `tickets.first_response_at - tickets.created_at`
   - Timestamps are `TIMESTAMPTZ`

3. Schema-link: tickets, ticket_priorities. Generate SQL:

   SELECT AVG(EXTRACT(EPOCH FROM (t.first_response_at - t.created_at)) / 3600)
          AS avg_response_hours
   FROM tickets t
   JOIN ticket_priorities tp ON t.priority_id = tp.id
   WHERE tp.name = 'Critical'
     AND t.created_at >= '2026-01-01';

4. Execute, validate, store full trace for future reuse.

Output:
| avg_response_hours |
|--------------------|
| 2.34               |
```

**Example 3: Multi-step analytical query with trace reuse**

```
User: "Which product categories have declining month-over-month revenue
for 3+ consecutive months?"
(Database: retail analytics, BigQuery)

Approach:
1. Retrieve prior trace for "monthly revenue by category" on this database.
   Exploration phase reveals:
   - Revenue = `order_items.quantity * order_items.unit_price`
   - Category via `products.category_id -> categories.name`
   - Use `orders.order_date` for time bucketing

2. Construct multi-CTE query (the execution pattern from memory guides
   CTE structure):

   WITH monthly_rev AS (
     SELECT c.name AS category,
            DATE_TRUNC(o.order_date, MONTH) AS month,
            SUM(oi.quantity * oi.unit_price) AS revenue
     FROM order_items oi
     JOIN orders o ON oi.order_id = o.id
     JOIN products p ON oi.product_id = p.id
     JOIN categories c ON p.category_id = c.id
     GROUP BY 1, 2
   ),
   with_lag AS (
     SELECT *, LAG(revenue) OVER (PARTITION BY category ORDER BY month) AS prev_rev
     FROM monthly_rev
   ),
   declining AS (
     SELECT category, month,
            CASE WHEN revenue < prev_rev THEN 1 ELSE 0 END AS is_decline
     FROM with_lag
   ),
   streaks AS (
     SELECT category, month, is_decline,
            SUM(CASE WHEN is_decline = 0 THEN 1 ELSE 0 END)
              OVER (PARTITION BY category ORDER BY month) AS streak_group
     FROM declining
   )
   SELECT DISTINCT category
   FROM (
     SELECT category, streak_group, COUNT(*) AS consecutive_declines
     FROM streaks WHERE is_decline = 1
     GROUP BY 1, 2
     HAVING COUNT(*) >= 3
   );

3. Execute, validate against known business context, store trace with
   "multi-month trend analysis" header for future similar questions.

Output:
| category     |
|--------------|
| Electronics  |
| Home & Garden|
```

## Best Practices

- **Do:** Store traces per-database with semantic headers. A trace for database A is useless for database B, but invaluable for the next question about A.
- **Do:** Separate exploration from execution in stored traces. Exploration phases are broadly reusable; execution phases are question-specific.
- **Do:** Use lightweight regex pattern matching (not LLM calls) to classify trace steps into phases. Reserve LLM calls for generating section summaries.
- **Do:** Bootstrap new databases with synthetic questions before handling real queries. One good synthetic trace prevents 5+ redundant exploration cycles.
- **Avoid:** Storing raw, unstructured execution logs. Unstructured traces cause "lost-in-the-middle" degradation and halve retrieval accuracy compared to structured markdown.
- **Avoid:** Delegating SQL generation to a separate sub-agent. Context loss during handoff degrades accuracy. Keep planning and SQL generation in a single agent.
- **Avoid:** Unlimited exploration budgets. Cap schema-linking to ~5 tool calls. Unbounded exploration wastes tokens and often loops without converging.

## Error Handling

- **Schema linking failures (30% of errors in enterprise settings):** When the agent selects wrong tables, fall back to broader vector search over the full schema. For nested schemas (e.g., Snowflake databases with multiple schemas), enumerate schema namespaces explicitly before linking.
- **SQL dialect mismatches:** If a query fails with syntax errors, check the target dialect. Common pitfalls: `DATE_TRUNC` argument order differs between PostgreSQL and BigQuery; Snowflake requires `ILIKE` for case-insensitive matching; BigQuery uses backtick-quoted table names.
- **Empty or unexpected results:** Re-examine filter conditions. Check if date columns use UTC vs. local time. Verify that JOIN conditions don't silently drop rows (switch to LEFT JOIN to diagnose).
- **Trace retrieval misses:** If no stored trace has high similarity, the agent must fall back to full exploration. This is expected for truly novel question types -- the trace generated here seeds memory for future queries.
- **Memory staleness:** If the database schema evolves (columns renamed, tables added), flag stored traces whose referenced tables/columns no longer exist. Invalidate stale traces rather than serving incorrect exploration context.

## Limitations

- **Single-database scoping:** Semantic memory retrieval filters by database. Cross-database joins or federated queries are not addressed by this technique.
- **Cold start cost:** The first query on a new database still requires full exploration (or synthetic bootstrapping), which is as expensive as a non-memory approach.
- **Schema evolution:** Stored traces become stale when schemas change. There is no built-in mechanism for automatic invalidation -- this must be managed externally.
- **Complex nested schemas:** Deeply nested schemas (e.g., Snowflake with multiple catalog levels) remain a significant source of errors even with memory, accounting for ~30% of failures in enterprise benchmarks.
- **Question diversity:** If real queries are highly diverse (touching entirely different parts of the schema each time), memory reuse benefits diminish. The technique shines when queries cluster around related table subsets.
- **Not a substitute for domain knowledge:** Semantic memory captures *how* to explore and query, not *what* business terms mean. Domain-specific glossaries or ontologies must be provided separately.

## Reference

Biswal, A., Lei, C., Qin, X., Li, A., & Narayanaswamy, B. (2026). *AgentSM: Semantic Memory for Agentic Text-to-SQL.* arXiv:2601.15709. https://arxiv.org/abs/2601.15709

Key takeaway: Section 3 (Semantic Memory Construction) and Section 4 (Trajectory Reading) detail how to structure, store, and retrieve execution traces. Table 1 shows the 2x accuracy gain of structured vs. unstructured trace formats. Algorithm 1 provides the synthetic question generation procedure for cold-start databases.