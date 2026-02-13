---
name: "benchmarking-text-to-python-against-text-to-sql"
description: "Generate correct Python/Pandas code from natural language questions over tabular data, applying the Logic Completion Framework (LCF) to resolve ambiguities that SQL handles implicitly. Use when: 'query this CSV with Python', 'convert this SQL to pandas', 'answer this question from the database using Python', 'write pandas code for this data question', 'translate SQL query to Python', 'analyze this table and answer questions about it'."
---

# Logic Completion Framework for Text-to-Python Data Querying

This skill enables Claude to generate reliable Python/Pandas code from natural language questions over tabular data by applying the Logic Completion Framework (LCF) from the BIRD-Python benchmark. The core insight: SQL databases silently handle NULL propagation, type coercion, case-insensitive matching, and aggregate semantics, but Python requires every one of these behaviors to be coded explicitly. When users ask data questions in natural language, the ambiguity that SQL resolves automatically becomes a source of bugs in Python. LCF systematically identifies and fills these gaps by injecting the missing domain logic before code generation.

## When to Use

- When a user asks a natural language question about data and expects Python/Pandas code (not SQL)
- When converting an existing SQL query to equivalent Pandas operations
- When a user loads a CSV, Parquet, or database table and asks analytical questions
- When building an analytical agent that must answer questions over file-based data using Python
- When debugging Pandas code that produces different results than the equivalent SQL query
- When a user's question is ambiguous about NULL handling, case sensitivity, or sort order and the answer must match SQL-standard behavior

## Key Technique

**The Paradigmatic Divergence Problem.** SQL is declarative: a `SELECT ... WHERE col = value` implicitly skips NULLs, coerces types, and applies database-default collation. Python is procedural: `df[df['col'] == value]` will include NaN comparisons (returning False but not filtering predictably), won't coerce `"42"` to `42`, and is case-sensitive by default. The BIRD-Python benchmark (Hu et al., 2026) demonstrates that most Text-to-Python failures stem not from faulty code generation but from these unspecified implicit behaviors. Models generate syntactically correct Pandas that silently produces wrong answers.

**The Logic Completion Framework (LCF)** addresses this by inserting an explicit logic resolution step before code generation. For every natural language question, LCF identifies which SQL-implicit behaviors are relevant (NULL handling, type casting, string collation, aggregation semantics, sort-order NULL placement) and specifies them as concrete constraints. For example, if a question asks "how many employees have a department?", LCF determines that this maps to `COUNT(department)` (excluding NULLs), not `COUNT(*)` (including all rows), and injects that specification into the prompt context. This transforms an ambiguous question into an unambiguous logical specification that the code generator can faithfully translate.

**Why it works.** The paper's experiments show that when these latent domain rules are made explicit, Text-to-Python achieves parity with Text-to-SQL across all tested LLMs. The gap was never about Python being harder to generate -- it was about missing context. This means any system generating Pandas from natural language can dramatically improve accuracy by systematically auditing for implicit SQL behaviors and resolving them before writing code.

## Step-by-Step Workflow

1. **Parse the data schema.** Read the table(s) involved -- column names, dtypes, sample rows. Identify columns with NULLs/NaNs, mixed types (strings that look numeric), and text columns that may need case-insensitive treatment.

2. **Decompose the natural language question into logical operations.** Map the question to a sequence of relational operations: filter, join, group, aggregate, sort, limit. Write these out in plain English before touching Pandas.

3. **Audit for implicit SQL behaviors.** For each operation identified in step 2, check whether any of these six categories apply:
   - **NULL handling**: Does filtering, counting, or joining involve columns with missing values? SQL's `WHERE col = X` silently excludes NULLs; Pandas does not.
   - **Type coercion**: Are you comparing or joining columns with mixed types (e.g., string `"42"` vs integer `42`)? SQL auto-casts; Pandas raises errors or silently mismatches.
   - **String collation**: Does the question involve name matching, search, or `LIKE`-style patterns? SQL defaults to case-insensitive in most databases; Pandas `str.contains()` is case-sensitive.
   - **Aggregation semantics**: Does `COUNT` mean count-all-rows or count-non-null? Does `AVG` skip NULLs? Pandas `.mean()` skips NaN by default (matching SQL), but `.count()` also skips NaN (matching `COUNT(col)`, not `COUNT(*)`).
   - **Sort NULL placement**: If sorting, where do NULLs go? Most SQL databases put NULLs last for ASC; Pandas puts them last by default too, but this must be verified.
   - **Duplicate handling**: Does the question imply DISTINCT? Pandas doesn't deduplicate unless told.

4. **Resolve each ambiguity with an explicit decision.** For each implicit behavior found, write a one-line specification: e.g., "Filter out rows where department is NaN before counting" or "Compare names case-insensitively". These are your logic completions.

5. **Generate the Pandas code with logic completions embedded.** Write the code so that each resolved behavior is visible as an explicit operation -- a `.dropna()`, a `.str.lower()`, a `na=False` parameter. Do not bury these in chained operations where they're invisible.

6. **Add defensive type handling.** Before comparisons or joins, explicitly cast columns to their expected types: `df['col'] = pd.to_numeric(df['col'], errors='coerce')` or `df['col'] = df['col'].astype(str).str.strip()`.

7. **Test against edge cases.** Mentally (or actually) verify: What happens if the filtered column is all NaN? What if the join key has type mismatches? What if the string column has leading/trailing whitespace?

8. **Return the result in the format the user expects.** If they asked a question, return the scalar answer. If they asked for a table, return the DataFrame. Include a brief comment explaining any non-obvious logic completions applied.

## Concrete Examples

**Example 1: Counting with NULL-sensitive semantics**

User: "How many employees have a manager assigned?"

```
Approach:
1. Parse schema: employees table has columns [id, name, manager_id, department].
   manager_id has NaN for top-level employees.
2. Logical operation: COUNT where manager_id is present.
3. Audit: NULL handling -- "have a manager assigned" means COUNT(manager_id), not COUNT(*).
4. Logic completion: Exclude rows where manager_id is NaN.

Code:
import pandas as pd

df = pd.read_csv("employees.csv")
# Logic completion: count only non-null manager_id values (matches SQL COUNT(manager_id))
result = df['manager_id'].notna().sum()
print(result)
```

Compare to the SQL equivalent that implicitly does this:
```sql
SELECT COUNT(manager_id) FROM employees;
```

**Example 2: Case-insensitive string filtering with type coercion**

User: "Find all products where the category contains 'electronics' and price is above 100"

```
Approach:
1. Parse schema: products table has [id, name, category (str), price (mixed: some stored as str)].
2. Logical operations: string LIKE filter on category + numeric comparison on price.
3. Audit:
   - String collation: "contains 'electronics'" should be case-insensitive (SQL LIKE default).
   - Type coercion: price column has some string values like "150.00" that need numeric cast.
   - NULL handling: skip rows where category or price is NaN.
4. Logic completions:
   - Use case-insensitive matching for category.
   - Coerce price to numeric, treating non-parseable values as NaN.
   - Exclude NaN rows from both filters.

Code:
import pandas as pd

df = pd.read_csv("products.csv")

# Logic completion: coerce price to numeric (SQL auto-casts string-to-number)
df['price'] = pd.to_numeric(df['price'], errors='coerce')

# Logic completion: case-insensitive match (SQL LIKE is case-insensitive by default)
# Logic completion: na=False excludes NaN rows (SQL WHERE skips NULLs)
mask = (
    df['category'].str.contains('electronics', case=False, na=False)
    & (df['price'] > 100)
)
result = df[mask]
print(result)
```

**Example 3: SQL-to-Pandas conversion with aggregation and sorting**

User: "Convert this SQL to Pandas:
`SELECT department, COUNT(*) as cnt FROM employees WHERE salary > 50000 GROUP BY department ORDER BY cnt DESC LIMIT 5;`"

```
Approach:
1. Decompose: filter -> group -> aggregate -> sort -> limit.
2. Audit:
   - NULL handling: WHERE salary > 50000 in SQL excludes NULLs implicitly. Pandas comparison
     with NaN returns False, which happens to match SQL here -- but only by accident.
   - COUNT(*) counts all rows in each group (including NULLs in other columns). Use .size(), not .count().
   - ORDER BY DESC with NULLs: not applicable here since count is always non-null.
3. Logic completions:
   - Explicitly drop NaN salary before filtering to match SQL WHERE behavior.
   - Use .size() for COUNT(*) semantics, not .count() which would give per-column non-null counts.

Code:
import pandas as pd

df = pd.read_csv("employees.csv")

# Logic completion: explicitly exclude NaN salary (matches SQL WHERE NULL exclusion)
filtered = df[df['salary'].notna() & (df['salary'] > 50000)]

# Logic completion: .size() matches COUNT(*) -- counts all rows per group
result = (
    filtered
    .groupby('department')
    .size()
    .reset_index(name='cnt')
    .sort_values('cnt', ascending=False)
    .head(5)
)
print(result)
```

## Best Practices

**Do:**
- Always check for NaN/NULL presence in every column involved in filters, joins, or aggregations before writing Pandas code. A 2-second `.isna().sum()` audit prevents silent wrong answers.
- Use `na=False` in all `.str.contains()`, `.str.startswith()`, and `.str.match()` calls. The default behavior (returning NaN for NaN inputs) causes filtering bugs.
- Use `.size()` for `COUNT(*)` semantics (count all rows in group) and `.count()` for `COUNT(col)` semantics (count non-null values). These are not interchangeable.
- Comment each logic completion in the generated code so the user understands why an explicit step exists (e.g., "# matches SQL case-insensitive LIKE behavior").

**Avoid:**
- Do not assume Pandas defaults match SQL defaults. They often coincidentally align (e.g., `.mean()` skips NaN like SQL `AVG`), but relying on coincidence is fragile. Be explicit.
- Do not chain long one-liners that hide NULL handling and type coercion inside nested method calls. Each logic completion should be a visible, separate line.
- Do not use `df['col'] == value` on columns with NaN without first deciding whether NaN rows should be included or excluded. The comparison returns `False` for NaN, which silently excludes them -- correct for SQL-style WHERE, but wrong if NaN is meaningful.
- Do not skip type coercion when reading CSVs. Pandas infers types, but columns with mixed content (e.g., "N/A" in a numeric column) become object dtype and break comparisons silently.

## Error Handling

| Problem | Symptom | Fix |
|---------|---------|-----|
| Mixed types in column | `TypeError` on comparison or silent wrong filter | Cast with `pd.to_numeric(errors='coerce')` or `.astype(str)` before comparing |
| NaN in join key | Unexpected row drops or duplications after merge | Filter NaN from join keys before merging: `df[df['key'].notna()]` |
| Case mismatch in string filter | Zero results when data exists | Add `case=False` to string methods, or `.str.lower()` both sides |
| COUNT(*) vs COUNT(col) confusion | Wrong aggregate counts | Use `.size()` for all-row count, `.count()` for non-null count |
| Whitespace in string columns | Join/filter misses on matching values | Apply `.str.strip()` before comparison |
| Pandas `.groupby()` drops NaN keys | Missing groups in output | Pass `dropna=False` to `.groupby()` if NULL groups matter |

## Limitations

- **Schema-dependent reasoning.** LCF requires knowing the data schema (column types, NULL patterns) before generating code. If the user provides only a question without schema context, ask to see the data or schema first.
- **Database-specific behaviors.** SQL behavior varies across databases (PostgreSQL sorts NULLs last by default; MySQL sorts them first for ASC). LCF resolves ambiguity to the most common convention, but if the user's SQL runs on a specific database, ask which one.
- **Complex nested subqueries.** Multi-level correlated subqueries with window functions are harder to decompose. The LCF approach still applies to each sub-operation, but the translation requires more careful step-by-step decomposition.
- **Performance at scale.** Pandas operates in-memory. For datasets exceeding available RAM, the Pandas translation is correct but may need chunked processing or a switch to Polars/Dask, which LCF does not directly address.
- **Non-relational operations.** LCF targets relational query semantics. Pandas operations with no SQL equivalent (e.g., time-series resampling, pivot tables beyond PIVOT) fall outside this framework.

## Reference

Hu, H., Hou, C., Cao, B., & Li, R. (2026). *Benchmarking Text-to-Python against Text-to-SQL: The Impact of Explicit Logic and Ambiguity.* arXiv:2601.15728v2. https://arxiv.org/abs/2601.15728v2

Key takeaway: The performance gap between Text-to-SQL and Text-to-Python is not about code generation quality -- it is about unresolved implicit logic. Systematically auditing for six categories of SQL-implicit behavior (NULLs, types, collation, aggregation, sort order, deduplication) and making them explicit closes the gap entirely.