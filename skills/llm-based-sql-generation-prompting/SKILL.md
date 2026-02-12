---
name: "llm-based-sql-generation-prompting"
description: "Generate accurate SQL from natural language using the SSEV pipeline: schema-linked prompting, execution-guided self-refinement, and weighted majority voting across multiple candidate queries. Use when the user says 'write SQL for this question', 'query this database', 'convert this question to SQL', 'text-to-SQL', 'generate a SQL query from natural language', or 'help me query this schema'."
---

# LLM-Based SQL Generation with Self-Refinement and Ensemble Voting

This skill enables Claude to convert natural language questions into accurate, executable SQL by applying the SSEV (Single-Agent Self-Refinement with Ensemble Voting) pipeline from Yang et al. (2026). Instead of generating a single SQL query and hoping it works, the technique generates multiple candidate queries using structured schema-aware prompts, refines failures through execution feedback, and selects the best result via weighted majority voting. This approach achieved 85.5% execution accuracy on Spider 1.0-Dev and 66.3% on BIRD-Dev without any ground-truth data.

## When to Use

- When the user provides a database schema (DDL, CREATE TABLE statements, or column lists) and asks a natural language question about the data
- When a generated SQL query fails with syntax errors or returns empty results and needs iterative fixing
- When the user needs SQL for a complex question involving multiple JOINs, subqueries, GROUP BY, or window functions
- When the user is working with an unfamiliar schema and needs help mapping their question to the right tables and columns
- When building a text-to-SQL pipeline or API that must handle diverse user questions reliably
- When the user needs SQL translated between dialects (MySQL, PostgreSQL, BigQuery, Snowflake)

## Key Technique: SSEV Pipeline

The core insight is a two-stage generate-then-refine pipeline with voting. **Stage 1 (PreSQL)** generates N candidate SQL queries from the full schema using a structured prompt containing four components: (1) task instructions with optimization rules, (2) full database DDL with foreign keys, (3) sample cell values (3 rows per table) to resolve formatting ambiguities, and (4) few-shot examples retrieved by embedding similarity. Each candidate is parsed to extract referenced tables and columns. The union of all extracted schema elements across candidates forms the "linked schema" -- the minimal relevant subset.

**Stage 2 (PostSQL)** rebuilds the prompt using only the linked schema, pruning irrelevant tables and columns. This reduces token noise and focuses attention. New candidates are generated from this tighter prompt. Any candidate that fails execution (syntax error or empty result) enters a **self-refinement loop**: the failed SQL, schema context, and database error message are re-prompted for up to N iterations until the query executes successfully.

**Voting** selects the final answer. Weighted Majority Voting (WMV) assigns each candidate generator a weight (initially uniform at 1.0). For each unique SQL string, the total weight is the sum of weights of generators that proposed it. The highest-weighted SQL wins. After each round, generators that produced incorrect results have their weights reduced by a multiplicative penalty: `w_i <- w_i * (1 - epsilon)` where `epsilon = sqrt(log(N) / T)`. This converges toward the most reliable generation strategy over multiple queries.

## Step-by-Step Workflow

1. **Parse the schema into DDL format.** Convert whatever schema representation the user provides (CREATE TABLE, JSON, column lists, ERD) into clean SQL DDL with table names, column names, data types, PRIMARY KEY constraints, and FOREIGN KEY relationships. This is the foundation of the prompt.

2. **Sample representative cell values.** If the user provides data or the database is accessible, extract 3 representative rows per table. Include these as INSERT or VALUES comments in the prompt. These resolve ambiguities like date formats (`2024-01-15` vs `Jan 15, 2024`), enum values (`M/F` vs `Male/Female`), and naming conventions.

3. **Construct the PreSQL prompt with four components:**
   - Task instruction: "Generate a SQL query that answers the following question. Minimize execution time. Use table aliases. Avoid SELECT *."
   - Full DDL: All CREATE TABLE statements with constraints
   - Cell values: Sample rows as comments
   - Few-shot examples: 2-3 similar question-SQL pairs if available

4. **Generate multiple candidate SQL queries.** Produce 3-5 candidate queries by varying the prompt framing (e.g., ask for the query directly, ask for step-by-step reasoning then query, ask for the simplest possible query). Use low temperature (~0.0) for each variant to ensure deterministic outputs from each prompt style.

5. **Extract the linked schema from candidates.** Parse all candidates with regex to find table and column references (FROM clauses, JOIN conditions, WHERE predicates, SELECT lists). Take the union of all referenced schema elements. This is the minimal relevant schema.

6. **Construct the PostSQL prompt with the pruned schema.** Replace the full DDL with only the linked tables and columns. Regenerate 3-5 candidates from this focused prompt. The pruned context reduces hallucinated column names and irrelevant JOINs.

7. **Execute each candidate and apply self-refinement.** Run each SQL against the database (or validate syntax if no database access). For any query that fails with a syntax error or returns empty results, re-prompt with: the failed SQL, the error message, the pruned schema, and the instruction "Fix this SQL query. The error was: [error]. Return only the corrected SQL." Retry up to 3 times.

8. **Select the final SQL via weighted voting.** Group candidates by normalized SQL string (ignore whitespace/case differences). Sum the weight of each generator backing each unique SQL. Return the SQL with the highest total weight. If there is a clear winner (>50% of total weight), present it with high confidence. If votes are split, present the top 2 candidates with explanations.

9. **Validate the result against the original question.** Check that the output columns match what was asked, required filters are present, aggregation matches the question intent (e.g., "how many" requires COUNT, "average" requires AVG), and no unnecessary columns are included.

10. **Present the SQL with an explanation.** Show the final query, explain the JOIN logic and WHERE conditions, and note any assumptions made about ambiguous terms in the question.

## Concrete Examples

**Example 1: Multi-table aggregation query**

User: "Given a database with tables `students(id, name, major_id, gpa)`, `majors(id, name, department)`, and `enrollments(student_id, course_id, grade)` -- how many students in each department have a GPA above 3.5?"

Approach:
1. Schema is already in DDL form. Identify FK: `students.major_id -> majors.id`
2. Generate PreSQL candidates:
   - Candidate A: `SELECT m.department, COUNT(*) FROM students s JOIN majors m ON s.major_id = m.id WHERE s.gpa > 3.5 GROUP BY m.department`
   - Candidate B: `SELECT department, COUNT(DISTINCT s.id) FROM students s INNER JOIN majors m ON s.major_id = m.id WHERE s.gpa > 3.5 GROUP BY department`
   - Candidate C: `SELECT m.department, COUNT(s.id) FROM majors m LEFT JOIN students s ON m.id = s.major_id AND s.gpa > 3.5 GROUP BY m.department`
3. Linked schema: `students(id, major_id, gpa)`, `majors(id, department)` -- `enrollments` pruned
4. PostSQL regeneration with pruned schema confirms candidates A and B
5. Candidates A and B are semantically equivalent (both INNER JOIN, both correct); C uses LEFT JOIN which includes departments with 0 qualifying students
6. Voting: A and B win (2 vs 1). Choose A for clarity.

Output:
```sql
SELECT m.department, COUNT(*) AS student_count
FROM students s
JOIN majors m ON s.major_id = m.id
WHERE s.gpa > 3.5
GROUP BY m.department;
```

**Example 2: Self-refinement on execution error**

User: "Using a `sales` table with columns `sale_date`, `amount`, `region`, find the month-over-month growth rate for each region."

Approach:
1. Generate initial candidate:
```sql
SELECT region, DATE_TRUNC('month', sale_date) AS month,
       SUM(amount) AS monthly_total,
       (SUM(amount) - LAG(SUM(amount)) OVER (PARTITION BY region ORDER BY DATE_TRUNC('month', sale_date)))
       / LAG(SUM(amount)) OVER (PARTITION BY region ORDER BY DATE_TRUNC('month', sale_date)) * 100 AS growth_pct
FROM sales
GROUP BY region, DATE_TRUNC('month', sale_date)
```
2. Execution fails: `window function LAG cannot be used in GROUP BY query without subquery` (dialect-dependent)
3. Self-refinement: Re-prompt with error. Fixed version uses a CTE:

Output:
```sql
WITH monthly AS (
  SELECT region,
         DATE_TRUNC('month', sale_date) AS month,
         SUM(amount) AS monthly_total
  FROM sales
  GROUP BY region, DATE_TRUNC('month', sale_date)
)
SELECT region, month, monthly_total,
       ROUND((monthly_total - LAG(monthly_total) OVER (PARTITION BY region ORDER BY month))
       / NULLIF(LAG(monthly_total) OVER (PARTITION BY region ORDER BY month), 0) * 100, 2) AS growth_pct
FROM monthly
ORDER BY region, month;
```

**Example 3: Ambiguous question requiring schema linking**

User: "Who are the top performers?" with schema `employees(id, name, department_id, hire_date)`, `performance_reviews(employee_id, review_date, score, reviewer_id)`, `sales_targets(employee_id, quarter, target, actual)`.

Approach:
1. "Top performers" is ambiguous -- could mean highest review scores or highest sales vs target
2. Generate candidates exploring both interpretations:
   - Candidate A: Uses `performance_reviews` with `ORDER BY AVG(score) DESC LIMIT 10`
   - Candidate B: Uses `sales_targets` with `ORDER BY (actual/target) DESC LIMIT 10`
   - Candidate C: Joins both tables for a composite score
3. Schema linking: Both `performance_reviews` and `sales_targets` are referenced -- cannot prune either
4. Present the ambiguity to the user and offer the most common interpretation:

Output:
```sql
-- Interpretation: Top performers by average review score
SELECT e.name, AVG(pr.score) AS avg_score, COUNT(pr.id) AS review_count
FROM employees e
JOIN performance_reviews pr ON e.id = pr.employee_id
GROUP BY e.id, e.name
HAVING COUNT(pr.id) >= 2
ORDER BY avg_score DESC
LIMIT 10;
```
> Note: "Top performers" could also refer to sales target achievement. If you meant sales performance, I can generate a query using `sales_targets.actual / sales_targets.target` instead.

## Best Practices

- **Do:** Always include sample cell values in your prompt context. A column named `status` could contain `"active"/"inactive"`, `1/0`, or `"A"/"I"` -- sample values resolve this.
- **Do:** Generate multiple candidates using varied prompt framings (direct, chain-of-thought, simplest-possible). Diversity in generation strategy catches different classes of errors.
- **Do:** Use the linked schema from PreSQL candidates to prune the PostSQL prompt. Removing irrelevant tables from the prompt significantly reduces hallucinated JOINs and phantom columns.
- **Do:** Apply NULLIF when dividing to prevent division-by-zero, and use COALESCE for NULL-safety in aggregations.
- **Avoid:** Generating a single SQL query and assuming it is correct. Even strong LLMs produce syntactically valid but semantically wrong SQL 15-30% of the time on complex schemas.
- **Avoid:** Including the entire database schema when only 2-3 tables are relevant. Token noise from irrelevant tables is the leading cause of incorrect table selection.
- **Avoid:** Skipping execution validation. Always attempt to run (or at least syntax-check) the generated SQL before presenting it as final.

## Error Handling

| Error Type | Detection | Recovery Strategy |
|---|---|---|
| Syntax Error | Database returns parse error | Re-prompt with error message and schema; fix specific syntax (missing comma, wrong keyword) |
| Table/Column Not Found | `relation X does not exist` | Check schema for correct names; apply fuzzy matching to find similar column names; re-link schema |
| Empty Result Set | Query returns 0 rows | Verify WHERE conditions aren't overly restrictive; check for NULL comparisons (use IS NULL instead of = NULL); relax filters |
| Type Mismatch | `operator does not exist: text = integer` | Cast columns explicitly; check if string values need quotes |
| Ambiguous Column | `column reference is ambiguous` | Add table aliases to all column references in JOINed queries |
| Timeout / Performance | Query runs too long | Add LIMIT for exploration; suggest indexes; rewrite correlated subqueries as JOINs |

When self-refinement fails after 3 attempts, present the best candidate with a clear explanation of what is failing and ask the user for clarification on the schema or question intent.

## Limitations

- **No database access:** Without the ability to execute queries, self-refinement is limited to syntax checking. Semantic correctness (does the query answer the right question?) cannot be verified.
- **Schema size:** Schemas with 50+ tables may exceed context limits even after pruning. For very large schemas, provide only tables the user mentions or that are obviously relevant.
- **Dialect differences:** SQL dialects vary significantly (DATE_TRUNC in PostgreSQL vs DATE_FORMAT in MySQL vs FORMAT in SQL Server). Always confirm the target dialect before generating.
- **Aggregation semantics:** Questions like "average revenue per customer" are ambiguous (per customer per month? lifetime?). When aggregation granularity is unclear, ask.
- **Domain knowledge:** Questions requiring business logic ("churned customers", "active users") depend on company-specific definitions that cannot be inferred from schema alone.
- **Complex nested queries:** Spider 2.0-Lite accuracy (31%) shows that enterprise-scale queries with multiple CTEs, cross-database joins, and dialect-specific functions remain challenging.

## Reference

Yang, Y.-J., Chang, H.-F., & Chen, P.-A. (2026). *LLM-Based SQL Generation: Prompting, Self-Refinement, and Adaptive Weighted Majority Voting.* arXiv:2601.17942v1. https://arxiv.org/abs/2601.17942v1

Key sections to consult: Section 3 for the SSEV pipeline architecture and PET-SQL prompt structure; Section 4 for the WMV/RWMA voting algorithms and weight update rules; Section 5 for the ReCAPAgent-SQL multi-agent framework with its seven specialized agents.