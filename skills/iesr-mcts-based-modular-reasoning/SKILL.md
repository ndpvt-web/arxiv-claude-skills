---
name: "iesr-mcts-based-modular-reasoning"
description: "Convert natural language questions into SQL queries using MCTS-based modular reasoning inspired by the IESR framework. Decomposes complex Text-to-SQL into information extraction, schema linking, multi-path tree search, and trajectory verification. Use when: 'generate SQL from this question', 'write a query for this database', 'translate this question to SQL', 'help me query this schema', 'complex SQL with calculations', 'text-to-SQL with reasoning'."
---

# IESR: MCTS-Based Modular Reasoning for Text-to-SQL

This skill enables Claude to convert complex natural language questions into accurate SQL queries by applying the IESR (Information Enhanced Structured Reasoning) framework. Instead of generating SQL in a single shot, Claude decomposes the problem into modular stages: extracting key information and constraints from the question, linking entities to the database schema, exploring multiple candidate SQL paths via a Monte Carlo Tree Search (MCTS) reasoning strategy, and verifying trajectory consistency across candidates. This approach excels on questions requiring mathematical computation, domain knowledge, unit conversions, or multi-step logical reasoning that single-pass generation gets wrong.

## When to Use

- When the user provides a natural language question and a database schema and asks for a SQL query
- When the question involves mathematical calculations, aggregations, or unit conversions that must be separated from the SQL structure
- When the schema is large or ambiguous and the correct tables/columns are not immediately obvious
- When the question requires multi-step reasoning (e.g., "Which store had the highest growth rate in Q3 compared to Q2?")
- When a first-attempt SQL query is wrong and the user wants a more rigorous reasoning approach
- When the user asks to query across multiple joined tables with complex filtering conditions
- When hypothetical or commonsense reasoning is needed (e.g., "Which products would expire within 30 days if purchased today?")

## Key Technique

**Modular Decomposition over Single-Shot Generation.** The core insight of IESR is that complex Text-to-SQL fails when treated as a monolithic translation task. Instead, the framework decomposes it into three stages: (1) Information Understanding -- extracting a latent semantic state from the question containing intent, entities, relations, numeric expressions, units, and field patterns; (2) Structured Reasoning -- using MCTS to explore multiple SQL construction paths through six heterogeneous actions (equation analysis, schema selection, column identification, entity extraction, SQL generation, and SQL revision); (3) Trajectory Verification -- scoring and selecting the best candidate using a composite of execution correctness, discriminator consistency, and peer agreement voting.

**MCTS as a Reasoning Engine.** Rather than beam search or simple self-consistency sampling, IESR uses Monte Carlo Tree Search where each node represents a partial SQL hypothesis with semantic context. The tree explores six action types that mirror how a human SQL expert thinks: first understand the math, then pick the right tables, then identify columns, then extract entities for filters, then write the SQL, then revise if needed. The UCT formula (Q(v,a)/N(v,a) + c * sqrt(ln(N(v))/N(v,a))) with exploration constant c=1.4 balances exploitation of promising paths with exploration of alternatives. Terminal nodes are scored by executing sampled SQL variants and measuring agreement rates.

**Mathematical Decoupling.** A critical innovation is treating mathematical computation as structurally orthogonal to SQL construction. Numeric expressions, unit conversions, and formulas are extracted and validated as separate reasoning concerns *before* SQL generation. This prevents the common failure mode where an LLM embeds incorrect arithmetic directly into a SQL expression.

## Step-by-Step Workflow

1. **Parse the question and schema.** Read the user's natural language question and database schema (DDL, table descriptions, or sample data). Identify all tables, columns, data types, primary keys, and foreign key relationships. If the schema is provided as text, convert it into a structured representation.

2. **Extract the latent semantic state.** From the question, explicitly extract:
   - **Intent**: What the user wants (count, list, compare, rank, aggregate)
   - **Entities**: Named values that will appear in WHERE clauses or JOINs
   - **Relations**: How entities connect across tables
   - **Numeric expressions**: Any math, formulas, or calculations mentioned
   - **Units**: Unit conversions or constraints (dates, currencies, measurements)
   - **Field patterns**: Column name hints from the question's phrasing

3. **Perform schema linking.** Map each extracted entity, relation, and field pattern to specific tables and columns in the schema. Resolve ambiguities by checking data types and foreign key paths. If multiple columns could match, keep all candidates for exploration.

4. **Decouple mathematical computation.** If the question involves calculations (growth rates, percentages, differences, averages over computed values), formalize the math separately. Write out the formula in plain terms, verify units are consistent, and determine whether the computation belongs in a SQL expression, a HAVING clause, or a subquery.

5. **Generate multiple candidate SQL paths.** Instead of writing one SQL query, generate 3-5 structurally different candidates by varying:
   - JOIN order and type (INNER vs LEFT)
   - Subquery vs CTE vs inline computation
   - WHERE clause structure (IN vs EXISTS vs JOIN filtering)
   - Aggregation strategy (GROUP BY placement, window functions vs subqueries)
   - How the mathematical computation is embedded

6. **Evaluate each candidate via execution reasoning.** For each candidate, mentally trace the execution: What rows would each JOIN produce? Does the WHERE clause filter correctly? Does the GROUP BY collapse the right groups? Does the ORDER BY/LIMIT return what was asked? Flag candidates with logical inconsistencies.

7. **Apply majority voting across candidates.** Compare the *structural intent* of surviving candidates. If 3 out of 5 candidates use the same JOIN path and aggregation strategy but differ only in syntax, that path is likely correct. Discard outlier approaches unless they address a flaw the majority misses.

8. **Verify trajectory consistency.** For the top candidate, walk through the reasoning chain: Does the schema linking justify every table in the FROM clause? Does every WHERE condition trace back to an extracted entity? Does the SELECT list answer the original intent? If any step is unjustified, revise.

9. **Produce the final SQL with explanation.** Output the SQL query along with a brief explanation mapping each clause back to the original question. Note any assumptions made about ambiguous terms.

10. **Offer revision if needed.** If the user reports incorrect results, use the "SQL Revision" action: diagnose which reasoning step failed (schema linking, math, filtering, grouping) and regenerate from that point rather than starting from scratch.

## Concrete Examples

**Example 1: Multi-step calculation with schema linking**

```
User: "Given the orders and products tables, which product category had the
highest average profit margin in 2024? Profit margin is (price - cost) / price."

Schema:
  orders(order_id, product_id, quantity, order_date)
  products(product_id, product_name, category, price, cost)

Approach:
1. Extract semantic state:
   - Intent: find the category with MAX average profit margin
   - Entities: year 2024
   - Numeric expression: (price - cost) / price
   - Relations: orders.product_id -> products.product_id

2. Decouple math: profit_margin = (price - cost) / price per product,
   then AVG(profit_margin) per category, then rank.

3. Schema linking: "product category" -> products.category,
   "2024" -> orders.order_date filter.

4. Generate candidates:
   Path A: JOIN + GROUP BY category with AVG((price-cost)/price)
   Path B: Subquery computing per-product margin, then outer GROUP BY
   Path C: CTE for margins, then aggregation

5. Majority vote: All three paths agree on the JOIN and GROUP BY structure.
   Path A is simplest and correct.

6. Verify: Every table is justified, the WHERE filters 2024, the formula
   matches the user's definition, ORDER BY DESC LIMIT 1 answers "highest."

Output:
SELECT p.category,
       AVG((p.price - p.cost) / p.price) AS avg_profit_margin
FROM orders o
JOIN products p ON o.product_id = p.product_id
WHERE o.order_date >= '2024-01-01'
  AND o.order_date < '2025-01-01'
ORDER BY avg_profit_margin DESC
LIMIT 1;
```

**Example 2: Hypothetical reasoning with commonsense**

```
User: "From the inventory table, which items would be out of stock within
7 days assuming daily sales stay at their current average?"

Schema:
  inventory(item_id, item_name, current_stock, warehouse_id)
  sales(sale_id, item_id, quantity, sale_date)

Approach:
1. Extract semantic state:
   - Intent: predict which items hit zero stock within 7 days
   - Numeric expression: current_stock / avg_daily_sales <= 7
   - Entities: "7 days", "current average" daily sales
   - This is hypothetical -- no future data exists, must compute from history

2. Decouple math:
   - avg_daily_sales = total_quantity_sold / number_of_distinct_sale_days
   - days_until_stockout = current_stock / avg_daily_sales
   - Filter: days_until_stockout <= 7

3. Schema linking: current_stock -> inventory.current_stock,
   daily sales -> derived from sales table grouped by item_id.

4. Generate candidates:
   Path A: CTE for avg daily sales, then JOIN with inventory, filter
   Path B: Subquery in WHERE clause
   Path C: Window function approach (rejected -- not applicable here)

5. Majority vote: Paths A and B agree structurally. Path A is clearer.

6. Verify: The CTE correctly computes per-item average daily sales.
   The JOIN links to inventory. The WHERE filters items running out in <= 7 days.
   Edge case: items with zero sales would cause division by zero -- add NULLIF.

Output:
WITH daily_avg AS (
    SELECT item_id,
           SUM(quantity) * 1.0 / COUNT(DISTINCT sale_date) AS avg_daily_sales
    FROM sales
    GROUP BY item_id
)
SELECT i.item_id,
       i.item_name,
       i.current_stock,
       da.avg_daily_sales,
       i.current_stock / da.avg_daily_sales AS days_until_stockout
FROM inventory i
JOIN daily_avg da ON i.item_id = da.item_id
WHERE da.avg_daily_sales > 0
  AND i.current_stock / da.avg_daily_sales <= 7
ORDER BY days_until_stockout ASC;
```

**Example 3: Ambiguous schema requiring multi-path exploration**

```
User: "How many users signed up each month but never made a purchase?"

Schema:
  users(id, name, email, created_at)
  accounts(account_id, user_id, signup_date, status)
  orders(order_id, account_id, total, created_at)

Approach:
1. Extract semantic state:
   - Intent: COUNT users per month who have zero purchases
   - Entities: "signed up" is ambiguous -- users.created_at or accounts.signup_date?
   - Relations: users -> accounts -> orders chain

2. Schema linking requires exploration:
   Path A: "signed up" = users.created_at (user registration)
   Path B: "signed up" = accounts.signup_date (account activation)
   Both are plausible. Explore both.

3. Generate candidates for each interpretation:
   Path A: LEFT JOIN orders via accounts, WHERE orders.order_id IS NULL,
           GROUP BY month of users.created_at
   Path B: LEFT JOIN orders directly on accounts, WHERE orders.order_id IS NULL,
           GROUP BY month of accounts.signup_date

4. Present both interpretations to the user with the SQL for each,
   noting the ambiguity.

Output (Path A -- user registration interpretation):
SELECT DATE_TRUNC('month', u.created_at) AS signup_month,
       COUNT(DISTINCT u.id) AS users_without_purchases
FROM users u
LEFT JOIN accounts a ON u.id = a.user_id
LEFT JOIN orders o ON a.account_id = o.account_id
WHERE o.order_id IS NULL
GROUP BY signup_month
ORDER BY signup_month;

-- Note: "signed up" could also refer to accounts.signup_date.
-- If so, replace DATE_TRUNC('month', u.created_at) with
-- DATE_TRUNC('month', a.signup_date) and adjust the base table.
```

## Best Practices

- **Do:** Always extract the semantic state (intent, entities, math, units) before writing any SQL. This prevents the most common failure mode of jumping to code with wrong assumptions.
- **Do:** Generate at least 2-3 structurally different candidate queries for complex questions. Single-shot generation misses alternative JOIN paths and aggregation strategies.
- **Do:** Decouple mathematical formulas from SQL syntax. Write out the math in plain language first, verify it, then embed it. This catches unit errors and formula mistakes early.
- **Do:** Explicitly flag ambiguous schema mappings and present alternatives to the user rather than silently picking one interpretation.
- **Avoid:** Generating SQL without reading and understanding the full schema first. Missing a foreign key relationship leads to incorrect JOINs.
- **Avoid:** Embedding complex arithmetic directly in SQL without first validating the formula separately. Nested expressions like `(a - b) / NULLIF(c, 0) * 100` are error-prone when composed inline.
- **Avoid:** Over-relying on a single candidate. If the question involves more than two JOINs or any calculation, always explore multiple paths before committing.

## Error Handling

| Failure Mode | Symptom | Recovery Strategy |
|---|---|---|
| Wrong schema linking | SQL references wrong table/column | Re-examine entities against schema; check foreign keys for the correct join path |
| Math error in SQL | Results are numerically wrong | Isolate the formula, compute a manual example, compare against SQL output |
| Missing JOIN | Fewer rows than expected or duplicates | Trace the entity-relation chain; add missing intermediate table |
| Ambiguous question | Multiple valid SQL interpretations | Present candidates for each interpretation; ask user to clarify |
| Division by zero | Runtime error on computed columns | Wrap denominators in NULLIF(expr, 0) or add WHERE clause filtering zeros |
| Date boundary error | Off-by-one in date ranges | Use >= start AND < next_period instead of BETWEEN for date ranges |
| GROUP BY mismatch | Aggregation error or wrong granularity | Verify every non-aggregated SELECT column appears in GROUP BY |

## Limitations

- **Requires schema access.** This approach needs the actual database schema (tables, columns, types, relationships). It cannot generate reliable SQL from vague descriptions like "the customer database."
- **No actual execution.** Claude cannot execute SQL against a live database to verify results. The MCTS-inspired multi-path approach simulates execution reasoning but cannot replace real testing. Always run generated SQL in a sandbox first.
- **Dialect differences.** Generated SQL targets standard SQL. Functions like `DATE_TRUNC`, `STRFTIME`, `DATEDIFF`, and string operations vary across MySQL, PostgreSQL, SQLite, and SQL Server. Specify your dialect upfront.
- **Very large schemas.** When a database has 50+ tables, schema linking becomes harder without access to data samples or documentation. Provide the relevant subset of the schema when possible.
- **Deeply nested logic.** Questions requiring 4+ levels of subquery nesting or recursive CTEs may exceed what multi-path reasoning can reliably verify without execution feedback.

## Reference

**Paper:** [IESR: Efficient MCTS-Based Modular Reasoning for Text-to-SQL with Large Language Models](https://arxiv.org/abs/2602.05385v1) (Liu et al., 2026). Key sections: Section 3 for the three-stage framework architecture, Section 3.2 for the six MCTS action types, Section 3.3 for the trajectory consistency scoring formula (Score(t) = 0.4*Exec + 0.2*DiscConf + 0.4*ConsVote), and Table 2 for ablation results showing each module's contribution.

**Code:** [https://github.com/Ffunkytao/IESR-SLM](https://github.com/Ffunkytao/IESR-SLM) -- Reference implementation using Qwen2.5-Coder-7B with MCTS configuration (32 rollouts, exploration constant 1.4, max depth 8).