---
name: "sqlagent-learning-explore-before"
description: "Explore unfamiliar databases before writing SQL by building a local knowledge base of schema fragments, executable queries, and natural language descriptions. Uses MCTS-inspired schema traversal and dual-agent retrieval+generation. Trigger phrases: 'explore this database then query it', 'build SQL knowledge base', 'generate SQL for unfamiliar schema', 'text-to-SQL with exploration', 'help me understand this database and write queries', 'explore before generating SQL'."
---

# SQLAgent: Explore Before Generating SQL

This skill teaches Claude to tackle text-to-SQL tasks on unfamiliar databases by first **exploring the database schema systematically** to build a knowledge base of working query patterns, then using that knowledge to generate accurate SQL for user questions. The technique comes from the SQLAgent paper (arXiv:2602.01952), which decouples knowledge acquisition from query generation using a Monte Carlo Tree Search-inspired exploration stage followed by a dual-agent retrieval-and-generation deployment stage. This is especially effective for complex, real-world databases where schemas are large, column names are ambiguous, and join paths are non-obvious.

## When to Use

- When the user provides a database (SQLite, PostgreSQL, MySQL) they want to query but the schema is unfamiliar, large, or poorly documented
- When the user asks Claude to "explore a database first" before writing queries
- When a text-to-SQL task involves multi-table joins with non-obvious foreign key paths
- When column names are ambiguous (e.g., multiple tables have `id`, `name`, `status` columns) and the user needs help disambiguating
- When the user wants to build a reusable set of query examples for a specific database
- When generating SQL for a domain-specific database with unusual naming conventions (e.g., medical, financial, or legacy enterprise schemas)
- When a single query attempt fails and the user needs an iterative refinement approach grounded in schema understanding

## Key Technique

**The core insight is: don't generate SQL cold.** Most text-to-SQL failures happen because the generator lacks familiarity with the specific database -- it doesn't know which tables connect, what values columns actually contain, or which join paths produce valid results. SQLAgent solves this by running an upfront exploration phase that produces a knowledge base of *tested, working* query patterns before any user question is answered.

**Exploration Stage (MCTS-Inspired).** The system navigates the database schema like a search tree. Each node represents a schema fragment (a subset of tables, columns, and joins). Using a strategy inspired by Monte Carlo Tree Search, it balances *exploitation* (deepening knowledge of promising table clusters) with *exploration* (visiting unexamined parts of the schema). At each node, it generates a candidate SQL query, executes it against the real database to verify it works, and if successful, records a **triplet**: (schema fragment used, executable SQL query, natural language description of what the query does). Failed queries provide negative signal that steers exploration away from invalid paths.

**Deployment Stage (Dual-Agent Retrieval + Generation).** When the user asks a question, a Retriever agent embeds the question and searches the knowledge base for semantically similar triplets. These retrieved examples are passed as in-context demonstrations to a Generator agent, which produces SQL grounded in *proven patterns* from the same database. If the generated SQL fails execution, the error message is fed back and the Generator retries with error context, up to a configurable number of iterations. This retrieval-augmented approach means the Generator never works from schema alone -- it always has working examples to anchor its output.

## Step-by-Step Workflow

### Phase 1: Exploration (Build the Knowledge Base)

1. **Connect and extract the full schema.** Read all table names, column names with data types, primary keys, foreign keys, and any constraints. Store this as a structured object (JSON or dictionary). For each table, also sample 3-5 rows to understand actual data patterns and value distributions.

2. **Identify table clusters via foreign keys.** Build a graph where nodes are tables and edges are foreign key relationships. Identify connected components and hub tables (tables with many foreign key references). These clusters define the natural join neighborhoods of the database.

3. **Run MCTS-inspired schema traversal.** For each table cluster, systematically explore query patterns:
   - **Select** a starting table (prefer hub tables or tables with the most columns first).
   - **Expand** by adding joins to neighboring tables one at a time through foreign key paths.
   - **Simulate** by generating a candidate SQL query using the current schema fragment -- include SELECT with representative columns, WHERE with plausible filters based on sampled data, and aggregations (COUNT, SUM, AVG) where numeric columns exist.
   - **Validate** by executing the query against the database. Record success/failure.
   - **Backpropagate** by tracking which schema fragments and join paths produce valid, non-empty results. Prioritize expanding fragments with high success rates while still exploring untested paths.

4. **Generate knowledge triplets.** For each successfully executed query, record three things:
   - The **schema fragment**: which tables and columns were involved, including the join path.
   - The **SQL query**: the exact, validated query text.
   - The **NL description**: a natural language sentence describing what the query computes (e.g., "Count of orders per customer in the last 30 days, joined through customer_id").

5. **Store the knowledge base.** Save all triplets in a searchable format (JSON array or SQLite table). Include metadata: tables touched, join depth, query complexity (number of joins, subqueries, aggregations).

### Phase 2: Deployment (Answer User Questions)

6. **Parse the user's natural language question.** Extract key entities, intent (lookup, aggregation, comparison, ranking), and any mentioned table/column names or domain terms.

7. **Retrieve relevant triplets.** Search the knowledge base for triplets whose NL descriptions or schema fragments are semantically similar to the user's question. Select the top 3-5 matches, preferring triplets that cover the same tables or similar join patterns.

8. **Generate SQL with retrieved context.** Construct a prompt that includes: the database schema, the retrieved triplets as few-shot examples, and the user's question. Generate the SQL query, using the examples to guide correct table selection, join paths, and column references.

9. **Execute and validate.** Run the generated SQL against the database. If it succeeds, return the result. If it fails, capture the error message.

10. **Iterative refinement (up to 3 attempts).** On failure, feed the error message and the failed query back into the generation step. The error context helps the Generator correct syntax errors, fix wrong column references, or adjust join conditions. After 3 failures, report the issue with diagnostic details rather than guessing further.

## Concrete Examples

**Example 1: Exploring a University Database**

```
User: I have a SQLite database at ./university.db. Explore it and then
      tell me which professors teach the most cross-department courses.

Approach:
1. Connect to university.db, extract schema:
   - tables: professors, courses, departments, teaching_assignments, enrollments
   - key FKs: teaching_assignments.professor_id -> professors.id,
              teaching_assignments.course_id -> courses.id,
              courses.department_id -> departments.id,
              professors.department_id -> departments.id

2. Exploration phase generates triplets like:
   Triplet A:
     Schema: professors JOIN teaching_assignments JOIN courses
     SQL: SELECT p.name, COUNT(DISTINCT c.department_id) as dept_count
          FROM professors p
          JOIN teaching_assignments ta ON p.id = ta.professor_id
          JOIN courses c ON ta.course_id = c.id
          GROUP BY p.id
     NL: "Count of distinct departments each professor teaches in"

   Triplet B:
     Schema: professors JOIN departments (self-department)
     SQL: SELECT p.name, d.name as home_dept
          FROM professors p JOIN departments d ON p.department_id = d.id
     NL: "Each professor with their home department name"

3. For the user's question, retrieve Triplets A and B as context.
   Generate final SQL:

   SELECT p.name, d.name AS home_department,
          COUNT(DISTINCT c.department_id) AS departments_taught
   FROM professors p
   JOIN departments d ON p.department_id = d.id
   JOIN teaching_assignments ta ON p.id = ta.professor_id
   JOIN courses c ON ta.course_id = c.id
   WHERE c.department_id != p.department_id
   GROUP BY p.id, p.name, d.name
   ORDER BY departments_taught DESC
   LIMIT 10;

Output: Table showing top 10 professors ranked by number of outside
        departments they teach courses in.
```

**Example 2: E-Commerce Database with Ambiguous Columns**

```
User: I have a Postgres database for our e-commerce platform. Help me
      find which product categories have the highest return rate.

Approach:
1. Extract schema and discover ambiguity:
   - tables: products, categories, orders, order_items, returns
   - Both orders and returns have a "status" column
   - Both products and categories have a "name" column
   - returns.order_item_id -> order_items.id -> orders.id

2. Exploration discovers the valid join path:
   returns -> order_items -> products -> categories
   (Not returns -> orders, which misses the product link)

   Triplet generated during exploration:
     Schema: returns JOIN order_items JOIN products JOIN categories
     SQL: SELECT c.name AS category, COUNT(r.id) AS return_count
          FROM returns r
          JOIN order_items oi ON r.order_item_id = oi.id
          JOIN products p ON oi.product_id = p.id
          JOIN categories c ON p.category_id = c.id
          GROUP BY c.id, c.name
     NL: "Return count per product category via order_items join path"

3. Retrieve this triplet and extend it to compute return *rate*:

   SELECT c.name AS category,
          COUNT(r.id) AS returns,
          COUNT(DISTINCT oi_all.id) AS total_items_sold,
          ROUND(COUNT(r.id)::numeric / NULLIF(COUNT(DISTINCT oi_all.id), 0) * 100, 2)
            AS return_rate_pct
   FROM categories c
   JOIN products p ON p.category_id = c.id
   JOIN order_items oi_all ON oi_all.product_id = p.id
   LEFT JOIN returns r ON r.order_item_id = oi_all.id
   GROUP BY c.id, c.name
   ORDER BY return_rate_pct DESC;
```

**Example 3: Iterative Refinement on Error**

```
User: How many active users logged in each month this year?

Approach:
1. Retrieved triplet suggests: SELECT MONTH(login_date) ...
2. Execute -> Error: "function month does not exist" (PostgreSQL)
3. Refinement: Replace MONTH() with EXTRACT(MONTH FROM login_date)
   and DATE_TRUNC('month', login_date) for grouping.
4. Execute -> Success. Return results.

The error-driven refinement replaces dialect-specific functions
automatically using the actual database engine's error feedback.
```

## Best Practices

- **Do:** Always execute exploratory queries against the real database during the exploration phase. Untested queries in the knowledge base are worse than no knowledge base at all -- they teach wrong patterns.
- **Do:** Sample actual data values from columns during exploration. Knowing that `status` contains `'active'`, `'suspended'`, `'deleted'` prevents generating queries with invented filter values like `'enabled'`.
- **Do:** Record failed queries during exploration as negative examples. If `JOIN orders ON products.id = orders.id` fails (wrong FK path), this prevents the Generator from repeating the mistake.
- **Do:** Qualify all column references with table aliases in generated SQL. This eliminates ambiguity errors on databases where multiple tables share column names.
- **Avoid:** Skipping the exploration phase and jumping straight to query generation on unfamiliar databases. The whole point is that cold generation fails on complex schemas.
- **Avoid:** Exploring exhaustively on very large schemas (100+ tables). Instead, focus exploration on the table cluster relevant to the user's domain. Use the user's question keywords to seed which tables to explore first.
- **Avoid:** Storing more than 50-100 triplets per table cluster. Diminishing returns set in quickly; prefer diverse query patterns over many similar ones.

## Error Handling

| Error | Cause | Resolution |
|-------|-------|------------|
| Ambiguous column reference | Multiple tables have same column name | Add explicit table alias qualification; check retrieved triplets for correct table |
| Invalid join path | Foreign key assumption was wrong | Re-examine schema FKs; try alternative join paths discovered during exploration |
| Empty result set | Filters too restrictive or wrong column values | Sample the filtered columns to verify actual values; loosen WHERE conditions |
| SQL dialect mismatch | Using MySQL syntax on PostgreSQL or vice versa | Check database engine type first; use engine-specific functions (e.g., `EXTRACT` vs `MONTH()`, `LIMIT` vs `TOP`) |
| Timeout on large tables | Query scans full table without index | Add WHERE clauses to limit scope; suggest the user add an index if this is a recurring pattern |
| Exploration phase produces few valid triplets | Schema has unusual structure or limited data | Fall back to schema-only generation; note to user that confidence is lower without exploration data |

## Limitations

- **Requires database access.** The exploration phase needs to execute queries against the real database. Read-only access is sufficient, but without any access, the technique degrades to schema-only generation.
- **Cold start on very large schemas.** Databases with hundreds of tables require scoping exploration to relevant subsets. Full exploration is impractical and unnecessary.
- **Not a replacement for domain knowledge.** If column semantics are deeply domain-specific (e.g., medical codes, financial instrument types), exploration can discover patterns but may not capture business logic that isn't encoded in the schema.
- **Single-database focus.** The knowledge base is specific to one database. Cross-database queries or federated schemas require separate exploration per source.
- **Iterative refinement has limits.** If the schema genuinely cannot answer the user's question (missing data), no amount of refinement will produce a valid query. Detect this and report it clearly.

## Reference

- **Paper:** [SQLAgent: Learning to Explore Before Generating as a Data Engineer](https://arxiv.org/abs/2602.01952v1) -- Look for the MCTS exploration algorithm, triplet generation strategy, and the dual-agent retrieval/generation architecture in the Deployment stage.