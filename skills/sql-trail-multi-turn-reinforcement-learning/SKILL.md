---
name: "sql-trail-multi-turn-reinforcement-learning"
description: "Iterative multi-turn Text-to-SQL generation using reason-execute-observe loops with execution feedback. Instead of writing SQL in one shot, explores the schema, executes intermediate queries, reads error messages and result previews, and refines until correct. Use when: 'write a SQL query for this database', 'help me query this schema', 'debug this SQL query', 'convert this question to SQL', 'analyze this database and answer', 'fix my failing SQL'."
---

# SQL-Trail: Multi-Turn Iterative Text-to-SQL Generation

This skill enables Claude to generate SQL queries through an iterative reason-execute-observe loop rather than producing a single-shot answer. Inspired by the SQL-Trail framework (arXiv:2601.17699), the approach mirrors how human SQL experts actually work: explore the schema, run a probe query, inspect results, spot mistakes, and refine. This is especially effective for complex joins, ambiguous column names, nested subqueries, and questions that require understanding the actual data distribution before writing a correct query.

## When to Use

- When a user provides a database schema and a natural language question to convert to SQL
- When a user pastes a failing SQL query and asks for help debugging it
- When the question involves multiple tables, ambiguous joins, or unclear column semantics
- When the user has a live database connection and wants verified, executable SQL
- When the question is complex enough that a single-shot query is likely to have errors (nested aggregations, CTEs, window functions, conditional logic)
- When the user asks to explore a database schema to understand what data is available before querying
- When prior SQL attempts returned wrong results and the user needs iterative correction

## Key Technique

**Single-shot SQL generation fails on hard queries.** Traditional approaches produce one query and hope it's correct. SQL-Trail replaces this with a multi-turn agentic loop: the agent reasons about the question, writes a SQL query, executes it against the database, reads the results or error message, and uses that feedback to refine its next attempt. This mirrors how a human DBA works — probing schema, testing hypotheses, and correcting mistakes iteratively.

**Two mechanisms make this efficient.** First, an adaptive turn budget scales interaction depth to question difficulty: simple lookups should resolve in 1-2 turns, while complex analytical queries get more room to explore. The agent should not waste turns on easy questions or give up too early on hard ones. Second, a composite evaluation balances multiple objectives: SQL correctness (does the output match?), turn efficiency (did we solve it concisely?), schema grounding (did we use the right tables and columns?), and syntactic validity (does the query parse?).

**The core loop is reason-execute-observe.** At each turn the agent: (1) reasons in natural language about what to try and why, (2) produces a SQL query, (3) receives execution output (results, errors, or truncated dataframe previews with column headers), and (4) appends this observation to its context before the next turn. The loop terminates when the agent is confident in a final answer or the turn budget is exhausted.

## Step-by-Step Workflow

1. **Parse the question and schema.** Read the user's natural language question. Identify the database schema — table names, column names, data types, primary/foreign keys, and any provided DDL or sample data. If the schema is not provided, ask for it or explore it with `INFORMATION_SCHEMA` queries.

2. **List relevant tables and columns before writing any SQL.** Explicitly enumerate which tables and columns are likely needed to answer the question. This schema grounding step prevents hallucinating column names and catches ambiguous references early.

3. **Write an initial probe query.** Start with a simpler version of the target query — perhaps querying a single table, checking a `SELECT *` with `LIMIT 5` to verify column names and data formats, or testing one join at a time. Do not attempt the full complex query on the first turn.

4. **Execute the probe and read the observation.** Run the query against the database. Inspect the results carefully: check column headers match expectations, verify data types, note NULL patterns, confirm join cardinality, and read any error messages fully.

5. **Reason about what the observation tells you.** Did the probe return expected results? Are there unexpected NULLs, duplicated rows from a bad join, or a data format that changes the approach (e.g., dates stored as strings)? Write this reasoning explicitly before the next query.

6. **Iteratively refine toward the final query.** Build up complexity incrementally — add joins, filters, aggregations, and subqueries one at a time. After each execution, verify intermediate results make sense. Fix errors by reading the error message and adjusting (wrong column name, type mismatch, missing GROUP BY, etc.).

7. **Apply the adaptive turn budget.** For simple lookups (single table, direct filter), aim to finish in 1-2 turns. For medium queries (1-2 joins, basic aggregation), use 2-3 turns. For hard queries (multiple joins, subqueries, window functions, CTEs), allow up to 5-6 turns. If you are not converging, step back and reconsider the approach rather than making small tweaks indefinitely.

8. **Validate the final query before presenting it.** Execute the complete query one last time. Verify the output columns match what the question asks for. Check that the result count is reasonable. Ensure the `SELECT` clause only includes columns explicitly requested by the user.

9. **Present the final SQL with explanation.** Give the user the final query, a brief explanation of the approach (which tables were joined and why, what filters were applied), and the execution result. If any assumptions were made about ambiguous terms, state them clearly.

10. **Offer to refine further.** If the user says the results look wrong, re-enter the loop: examine the discrepancy, run diagnostic queries, and adjust.

## Concrete Examples

**Example 1: Multi-table aggregation with ambiguous schema**

```
User: "Using the company database, find the top 5 departments by average salary,
but only include departments with more than 10 employees."

Schema provided: employees(id, name, dept_id, salary, hire_date),
                 departments(id, dept_name, location)

Approach:

Turn 1 - Schema grounding and probe:
  Reasoning: Need to join employees to departments on dept_id = departments.id.
  First, verify the join works and check data.
  Query: SELECT d.dept_name, COUNT(*) as cnt, AVG(e.salary) as avg_sal
         FROM employees e JOIN departments d ON e.dept_id = d.id
         GROUP BY d.dept_name LIMIT 5;
  Observation: Returns 5 rows with dept_name, cnt, avg_sal. Data looks correct.
  Counts range from 3 to 45. avg_sal values are reasonable.

Turn 2 - Add filter and ordering:
  Reasoning: Probe confirmed the join. Now add HAVING > 10 and ORDER BY.
  Query: SELECT d.dept_name, COUNT(*) as employee_count,
                ROUND(AVG(e.salary), 2) as avg_salary
         FROM employees e JOIN departments d ON e.dept_id = d.id
         GROUP BY d.dept_name
         HAVING COUNT(*) > 10
         ORDER BY avg_salary DESC
         LIMIT 5;
  Observation: Returns 5 rows. Engineering: $142,300, Marketing: $128,750, ...

Final output: Verified query with results presented to user.
```

**Example 2: Debugging a failing query with error feedback**

```
User: "This query keeps failing, can you fix it?"
  SELECT customer_name, SUM(order_total)
  FROM orders JOIN customers ON orders.cust_id = customers.id
  WHERE order_date > '2024-01-01'
  GROUP BY customer_id;

Approach:

Turn 1 - Execute the broken query as-is:
  Observation: ERROR: column "customer_name" must appear in the GROUP BY
  clause or be used in an aggregate function.

Turn 2 - Reason and fix:
  Reasoning: GROUP BY uses customer_id but SELECT includes customer_name.
  Need to GROUP BY both, or GROUP BY customers.id since customer_name is
  functionally dependent. Also, customer_id is ambiguous — should specify
  table. Let me also verify column names exist.
  Query: SELECT c.customer_name, SUM(o.order_total) as total_spent
         FROM orders o JOIN customers c ON o.cust_id = c.id
         WHERE o.order_date > '2024-01-01'
         GROUP BY c.id, c.customer_name
         ORDER BY total_spent DESC;
  Observation: Returns 47 rows. Looks correct.

Final output: Fixed query with explanation of the GROUP BY issue and
table alias best practice.
```

**Example 3: Complex analytical query requiring exploration**

```
User: "Find customers who made purchases in every month of 2024."

Schema: orders(id, customer_id, order_date, total), customers(id, name, email)

Approach:

Turn 1 - Explore the data range:
  Reasoning: Need to verify 2024 data exists and understand date format.
  Query: SELECT MIN(order_date), MAX(order_date),
                COUNT(DISTINCT EXTRACT(MONTH FROM order_date))
         FROM orders WHERE order_date >= '2024-01-01'
                       AND order_date < '2025-01-01';
  Observation: min=2024-01-03, max=2024-12-28, count=12. All 12 months present.

Turn 2 - Build the core logic:
  Reasoning: Need customers where COUNT(DISTINCT month) = 12 within 2024.
  Query: SELECT c.name, COUNT(DISTINCT EXTRACT(MONTH FROM o.order_date)) as months
         FROM orders o JOIN customers c ON o.customer_id = c.id
         WHERE o.order_date >= '2024-01-01' AND o.order_date < '2025-01-01'
         GROUP BY c.id, c.name
         HAVING COUNT(DISTINCT EXTRACT(MONTH FROM o.order_date)) = 12;
  Observation: Returns 8 customers. Spot-check one with a drill-down next.

Turn 3 - Validate a sample result:
  Reasoning: Verify customer "Alice Chen" actually has orders in all 12 months.
  Query: SELECT EXTRACT(MONTH FROM order_date) as mo, COUNT(*)
         FROM orders WHERE customer_id = 42
           AND order_date >= '2024-01-01' AND order_date < '2025-01-01'
         GROUP BY mo ORDER BY mo;
  Observation: 12 rows, months 1-12 all present. Confirmed.

Final output: Verified query with 8 results and validation methodology explained.
```

## Best Practices

- **Do:** Always run a schema probe first on unfamiliar databases. A quick `SELECT * FROM table LIMIT 5` reveals actual column names, data types, and NULL patterns that the DDL alone may not make obvious.
- **Do:** Build complex queries incrementally. Test each join, filter, and aggregation separately before combining. This isolates errors to the specific component that introduced them.
- **Do:** Read error messages and result previews carefully before the next attempt. The observation is the most valuable signal — do not ignore unexpected NULLs, row counts, or column values.
- **Do:** State your reasoning explicitly before each query. This prevents circular edits where you change something without understanding why the previous attempt failed.
- **Avoid:** Writing the full complex query on the first turn for anything beyond simple lookups. Single-shot attempts on hard queries waste turns on cascading errors.
- **Avoid:** Burning turns on cosmetic changes (aliasing, formatting) before the query logic is correct. Get correct results first, then polish.
- **Avoid:** Ignoring the turn budget. If you are on turn 5 and still not converging, step back and reconsider the fundamental approach (wrong join path, misunderstood question) rather than making incremental patches.

## Error Handling

| Error Type | What to Do |
|---|---|
| Column not found | Check actual column names with `INFORMATION_SCHEMA.COLUMNS` or `SELECT *` probe. Schema descriptions may be stale or abbreviated. |
| Ambiguous column reference | Add explicit table aliases to all column references. |
| Type mismatch (e.g., comparing string to int) | Inspect actual data types with a probe query. Apply `CAST()` or adjust the comparison. |
| Cartesian product / exploding row count | Check join conditions — a missing or incorrect ON clause causes cross joins. Verify with `COUNT(*)` before and after the join. |
| Empty result set | Verify filter values exist in the data. Run the query without filters to confirm base data exists, then add filters back one at a time. |
| Timeout on large tables | Add `LIMIT` to probe queries. Use indexed columns in WHERE clauses. Consider whether a subquery can reduce the working set. |
| Syntax varies by dialect | Ask which database engine (PostgreSQL, MySQL, SQLite, SQL Server, etc.) and adjust syntax accordingly (e.g., `LIMIT` vs `TOP`, string functions, date handling). |

## Limitations

- **No database connection:** If the user cannot execute queries (schema-only context), the iterative feedback loop is unavailable. Fall back to careful single-pass generation with explicit schema grounding and ask the user to run and report results.
- **Read-only feedback:** This approach relies on SELECT execution feedback. INSERT/UPDATE/DELETE operations cannot be safely probed iteratively without transaction rollback support.
- **Very large result sets:** Execution observations should be truncated (first 10-20 rows). Do not attempt to reason over thousands of result rows — instead, use aggregation queries to summarize.
- **Dialect-specific features:** Complex dialect-specific syntax (recursive CTEs, lateral joins, proprietary functions) may require knowing the exact database engine upfront.
- **Schema changes during session:** If the database schema is being modified concurrently, cached schema knowledge from earlier turns may become stale.

## Reference

**SQL-Trail: Multi-Turn Reinforcement Learning with Interleaved Feedback for Text-to-SQL** — arXiv:2601.17699 (2026). Key takeaway: iterative reason-execute-observe loops with adaptive turn budgets and composite reward signals (correctness + efficiency + schema grounding) outperform single-shot SQL generation by large margins, especially on complex queries. Look for Section 3 (method) and the composite reward formulation in Section 3.3.