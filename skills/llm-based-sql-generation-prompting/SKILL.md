---
name: "llm-based-sql-generation-prompting"
description: "Generate accurate SQL from natural language using SSEV pipeline techniques: schema linking, self-refinement loops, and weighted majority voting. Use when user says 'write SQL for', 'query this database', 'text-to-SQL', 'generate SQL from question', 'help me write a database query', or 'translate this question to SQL'."
---

# LLM-Based SQL Generation with Self-Refinement and Ensemble Voting

This skill enables Claude to generate high-accuracy SQL from natural language questions by applying the SSEV (Single-Agent Self-Refinement with Ensemble Voting) pipeline from Yang et al. (2026). Instead of generating SQL in a single pass, Claude follows a structured workflow: first linking the question to relevant schema elements, then generating candidate SQL, then iteratively self-refining against execution errors, and finally selecting the best candidate through weighted voting when multiple approaches exist. This approach achieved 85.5-86.4% execution accuracy on Spider benchmarks and 66.3% on BIRD, substantially outperforming single-shot generation.

## When to Use

- When the user provides a natural language question and a database schema and asks for a SQL query
- When a generated SQL query fails with execution errors and needs systematic debugging
- When the user has a complex schema with many tables and needs help identifying which tables and columns are relevant
- When the user wants SQL that works across dialects (MySQL, PostgreSQL, SQLite, BigQuery, Snowflake)
- When the user asks "why is my SQL query returning empty results" or "fix this SQL error"
- When building a text-to-SQL pipeline or API endpoint that needs high accuracy
- When the question involves JOINs across multiple tables, subqueries, or aggregations where ambiguity is high

## Key Technique: SSEV Pipeline

The core insight is that single-pass SQL generation fails on hard queries because of three compounding problems: schema ambiguity (which tables/columns does the question reference?), SQL complexity (nested subqueries, GROUP BY with HAVING, etc.), and execution correctness (syntactically valid SQL that returns wrong results). SSEV addresses all three through a staged pipeline.

**Schema Linking via PreSQL Parsing.** Rather than trying to match question words to schema elements directly (which is fragile), generate a rough "PreSQL" draft using the full schema, then parse that draft to extract which tables and columns the LLM actually referenced. Union the extracted elements across multiple drafts to build a minimal relevant schema. This pruned schema dramatically reduces noise in subsequent generation -- the LLM sees only what matters instead of hundreds of irrelevant columns.

**Self-Refinement with Error Classification.** When a generated query fails, don't just re-prompt blindly. Classify the error (SyntaxError, TableNotFound, ColumnNotFound, TypeMismatch, NoResult) and attach error-specific correction hints to the re-prompt. Include the original SQL, the execution error message, and the schema context. Cap refinement at 3 iterations to avoid loops. This targeted feedback loop resolves 15-25% of initially failing queries.

**Weighted Majority Voting.** When generating multiple candidate SQL queries (via different prompting strategies or temperature variation), don't pick randomly. Assign each strategy a weight (initially uniform), aggregate weights for each unique SQL output, and select the highest-weighted candidate. After observing correctness, penalize wrong strategies multiplicatively: `w_i <- w_i * (1 - epsilon)`. Over multiple queries in a session, the system learns which strategies are most reliable for this particular database.

## Step-by-Step Workflow

1. **Collect schema and question.** Obtain the full database DDL (CREATE TABLE statements with column types and foreign keys). If unavailable, inspect the database to reconstruct it. Also obtain 2-3 sample rows per table to understand value formats (date formats, enum values, naming conventions).

2. **Build the initial prompt.** Structure the prompt with four components in order: (a) task instruction including "generate executable SQL that minimizes execution time", (b) full DDL with foreign key annotations, (c) sample cell values (3 representative rows per table), (d) the natural language question. If available, include 2-3 similar question-SQL pairs as few-shot examples.

3. **Generate PreSQL candidates.** Produce 2-3 SQL candidates using different prompting variations: one with chain-of-thought reasoning ("think step by step about which tables are needed"), one direct, and one with explicit schema analysis ("first list the relevant tables and columns, then write the query"). Use moderate temperature (0.4-0.7) for diversity.

4. **Extract schema links from PreSQL.** Parse each candidate SQL to extract all table and column references using patterns like `FROM table`, `JOIN table`, `table.column`, `SELECT column`. Take the union of all referenced elements across candidates. This is your linked schema.

5. **Prune and regenerate (PostSQL).** Rebuild the prompt using only the linked schema elements instead of the full DDL. Regenerate SQL candidates with this focused context. The reduced noise typically improves accuracy by 3-8 percentage points.

6. **Execute and classify errors.** Run each candidate against the database. For each failure, classify the error:
   - `SyntaxError`: malformed SQL -- hint: check keyword spelling, parenthesis matching, dialect-specific syntax
   - `TableNotFound` / `ColumnNotFound`: wrong schema reference -- hint: re-examine available tables/columns, check for aliases
   - `TypeMismatch`: incompatible operations -- hint: add CAST expressions, check date/string formats
   - `NoResult`: empty result set -- hint: check WHERE conditions, verify JOIN keys match, consider NULL handling

7. **Self-refine failing candidates.** For each failed query, construct a refinement prompt containing: the original SQL, the execution error message, the error classification and its specific hint, and the pruned schema. Generate a corrected SQL. Repeat up to 3 iterations or until execution succeeds.

8. **Select the best candidate via weighted voting.** Group all surviving candidates (post-refinement) by their SQL text. If multiple candidates agree on the same SQL, that's a strong signal. For a single query, pick the candidate that executes successfully and returns non-empty results. For a batch of queries against the same database, maintain per-strategy weights and apply weighted majority voting.

9. **Validate the result.** Check that the output makes sense: correct number of columns for the question, non-empty where results are expected, reasonable row count. If the question asks "how many", ensure the result is a single number. If it asks "list all", ensure multiple rows.

10. **Return the SQL with explanation.** Present the final SQL with a brief explanation of the reasoning: which tables were joined and why, what the WHERE clause filters, and how aggregations map to the question.

## Concrete Examples

**Example 1: Multi-table JOIN with aggregation**

User: "Using the university database, find the average salary of professors in each department that has more than 5 professors."

Schema (provided or discovered):
```sql
CREATE TABLE department (dept_id INT PRIMARY KEY, dept_name VARCHAR(100));
CREATE TABLE professor (prof_id INT PRIMARY KEY, name VARCHAR(100), dept_id INT REFERENCES department(dept_id), salary DECIMAL(10,2));
```

Approach:
1. Schema link: question references "professors" -> `professor` table, "department" -> `department` table, "salary" -> `professor.salary`
2. Pruned schema: both tables fully relevant, no pruning needed
3. Generate candidate SQL with chain-of-thought:
   - Need GROUP BY department
   - HAVING count > 5 for filtering
   - JOIN to get department names

Output:
```sql
SELECT d.dept_name, AVG(p.salary) AS avg_salary
FROM professor p
JOIN department d ON p.dept_id = d.dept_id
GROUP BY d.dept_id, d.dept_name
HAVING COUNT(p.prof_id) > 5
ORDER BY avg_salary DESC;
```

**Example 2: Self-refinement after execution error**

User: "Show me all orders placed in the last 30 days with their customer names."

First attempt (PreSQL):
```sql
SELECT c.name, o.order_id, o.order_date
FROM orders o JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_date > DATE_SUB(CURRENT_DATE, INTERVAL 30 DAY);
```

Execution error: `ERROR: function date_sub does not exist` (PostgreSQL dialect)

Error classification: `SyntaxError` -- dialect mismatch, DATE_SUB is MySQL-specific.

Refinement prompt includes: original SQL + error message + hint "check dialect-specific date functions; PostgreSQL uses CURRENT_DATE - INTERVAL '30 days'"

Refined output:
```sql
SELECT c.name, o.order_id, o.order_date
FROM orders o JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_date > CURRENT_DATE - INTERVAL '30 days'
ORDER BY o.order_date DESC;
```

**Example 3: Complex schema linking on a large database**

User: "How many students got an A in courses taught by professors from the CS department?"

Schema: 8 tables including `student`, `enrollment`, `course`, `section`, `teaches`, `professor`, `department`, `grade_report`.

Approach:
1. Generate 3 PreSQL candidates with full 8-table schema
2. Parse candidates -- all reference `enrollment`, `course`, `teaches`, `professor`, `department`. None reference `student` directly (only need count). `grade_report` referenced by 2 of 3 candidates, `section` by 1.
3. Linked schema: `enrollment`, `course`, `teaches`, `professor`, `department`, `grade_report` (union)
4. Regenerate with pruned 6-table schema
5. Two candidates agree on the same JOIN path; one uses a subquery. Pick the majority.

Output:
```sql
SELECT COUNT(DISTINCT e.student_id)
FROM enrollment e
JOIN grade_report g ON e.enrollment_id = g.enrollment_id
JOIN section s ON e.section_id = s.section_id
JOIN teaches t ON s.section_id = t.section_id
JOIN professor p ON t.prof_id = p.prof_id
JOIN department d ON p.dept_id = d.dept_id
WHERE g.grade = 'A' AND d.dept_name = 'CS';
```

## Best Practices

**Do:** Always include sample cell values (2-3 rows per table) in your prompt. Value formats like `'2024-01-15'` vs `'01/15/2024'` or `'CS'` vs `'Computer Science'` are critical for correct WHERE clauses.

**Do:** Generate multiple candidates with varied prompting strategies (chain-of-thought, direct, schema-analysis-first) rather than relying on a single generation. Agreement between independent candidates is the strongest signal of correctness.

**Do:** Always specify the SQL dialect explicitly in the prompt ("Generate PostgreSQL-compatible SQL" or "Use SQLite syntax"). Dialect mismatches are a top-3 error category.

**Do:** When refining, include both the error message AND the original SQL in the prompt. The LLM needs to see what went wrong in context, not just the error.

**Avoid:** Sending the entire database schema when it has 50+ tables. Use the PreSQL schema linking step to prune first. Large schemas cause the LLM to hallucinate table/column names.

**Avoid:** Refining more than 3 times on the same query. If 3 iterations haven't fixed it, the problem is likely a fundamental misunderstanding of the question or schema, not a fixable SQL bug. Escalate to the user for clarification.

**Avoid:** Trusting a query just because it executes without error. Empty results, single NULL rows, or cartesian product explosions are all "successful" executions that indicate incorrect SQL.

## Error Handling

| Error Type | Detection | Recovery |
|---|---|---|
| Schema mismatch | `TableNotFound`, `ColumnNotFound` errors | Re-run schema linking; inspect actual table/column names with `SHOW TABLES` or `\dt`; check for typos and case sensitivity |
| Dialect incompatibility | Functions like `DATE_SUB`, `LIMIT` syntax, `ILIKE` not recognized | Identify target dialect, substitute equivalent functions, regenerate |
| Ambiguous question | Multiple valid interpretations produce different SQL | Present 2-3 interpretations to the user with their SQL and sample outputs, ask which is intended |
| Cartesian product | Query returns millions of rows unexpectedly | Check for missing JOIN conditions; every table in FROM should connect via a JOIN predicate |
| Empty results | Query returns 0 rows when results are expected | Check WHERE values against actual data; look for NULL comparisons (use `IS NULL` not `= NULL`); verify JOIN keys exist in both tables |
| Timeout | Query runs too long | Add indexes suggestion; simplify subqueries to CTEs; check for missing WHERE clauses causing full table scans |

## Limitations

- **No ground-truth feedback in single queries.** The weighted voting mechanism shines when processing batches of queries against the same database, where strategy weights can be tuned over time. For one-off queries, it degrades to simple majority voting.
- **Schema linking depends on PreSQL quality.** If the initial generation completely misidentifies the relevant tables, the pruned schema will be wrong, and all subsequent steps inherit that error.
- **Enterprise complexity ceiling.** On Spider 2.0-Lite (real-world enterprise databases with BigQuery/Snowflake), accuracy drops to ~31%. For production enterprise use, human review of generated SQL remains essential.
- **No learning across sessions.** The pipeline doesn't retain knowledge about a specific database's quirks. Each conversation starts fresh.
- **Requires execution access.** The self-refinement loop depends on running the SQL and observing errors. In read-only or no-execution contexts, skip steps 6-7 and rely on static analysis and candidate agreement instead.

## Reference

Yang, Y.-J., Chang, H.-F., & Chen, P.-A. (2026). *LLM-Based SQL Generation: Prompting, Self-Refinement, and Adaptive Weighted Majority Voting.* arXiv:2601.17942v1. https://arxiv.org/abs/2601.17942v1

Key sections to study: Section 3 for the SSEV pipeline architecture, Section 3.3 for Weighted Majority Voting algorithm details, Section 4 for ReCAPAgent-SQL's multi-agent design, and Figures 3-5 for prompt template structures.