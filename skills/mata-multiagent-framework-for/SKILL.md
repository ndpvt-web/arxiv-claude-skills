---
name: "mata-multiagent-framework-for"
description: "Multi-agent table question answering using MATA's three-path reasoning strategy (Chain-of-Thought, Program-of-Thought, Text-to-SQL) with confidence-based answer selection and scheduler-driven efficiency. Use when: 'answer questions about this table', 'analyze this CSV and answer', 'query this spreadsheet data', 'build a table QA pipeline', 'compare multiple reasoning approaches on tabular data', 'reliable table analysis with verification'."
---

# MATA: Multi-Agent Table Question Answering

This skill enables Claude to answer complex questions over tabular data by orchestrating three complementary reasoning paths — textual Chain-of-Thought (CoT), Python code generation (Program-of-Thought), and SQL query generation (Text-to-SQL) — then selecting the best answer through confidence scoring and optional judge arbitration. The key insight from MATA is that no single reasoning approach dominates across all table questions: numerical aggregation favors code, lookups favor SQL, and interpretive questions favor natural language reasoning. By generating candidates from all three paths and applying lightweight verification tools, you get far more reliable answers than any single method alone.

## When to Use

- When the user provides a table (CSV, JSON, markdown, DataFrame) and asks factual questions about its contents
- When answering questions that require arithmetic over table cells (sums, averages, counts, comparisons)
- When building a reliable table QA system that needs to handle diverse question types without knowing which will appear
- When the user wants to verify a table-derived answer by cross-checking multiple reasoning approaches
- When working with messy or large tables where a single SQL or Python approach might fail silently
- When designing an agent pipeline that needs to minimize expensive LLM calls while maintaining accuracy
- When the user asks to implement a multi-agent reasoning system for structured data

## Key Technique

MATA decomposes table question answering into three parallel reasoning paths. **Chain-of-Thought (CoT)** reasons directly in natural language — good for interpretive or lookup questions. **Program-of-Thought (PoT)** generates and executes Python code — strong for multi-step arithmetic and data manipulation. **Text-to-SQL (t2SA)** converts the question into a SQL query — efficient for filtering, grouping, and joining operations. Each path produces a candidate answer independently.

The framework then applies three lightweight tools to select the best answer without always needing an expensive judge LLM call. A **Scheduler** (a small classifier, ~25M parameters) examines the table structure and question to predict which code-based path (PoT or t2SA) is more likely to succeed, and runs that one first. If its answer agrees with CoT, the third path is skipped entirely — saving an LLM call. A **Confidence Checker** (~435M parameters, fine-tuned DeBERTa) scores each candidate answer's reliability. If any candidate exceeds a confidence threshold (theta=0.1), that answer is selected directly, bypassing the Judge Agent. Only when all candidates score low does a **Judge Agent** (another LLM call) arbitrate. In practice, the Confidence Checker resolves 60-96% of questions without needing the Judge, dramatically reducing inference cost.

The debug loop is also critical: code-based agents (PoT and t2SA) get up to N=3 iterative refinement attempts. When code fails or produces an error, a dedicated Debug Agent receives the error traceback and previous code, then generates a corrected version. Early stopping kicks in when consecutive iterations produce identical output.

## Step-by-Step Workflow

1. **Parse the input table into a structured format.** Convert CSV, markdown, or raw text into a pandas DataFrame (or equivalent). Identify column names, data types (numeric, string, date), row count, and any irregularities (merged cells, missing values, mixed types).

2. **Analyze the question to classify its type.** Determine whether the question involves lookup, arithmetic aggregation, comparison, ranking, filtering, or multi-step reasoning. This informs which reasoning path is most likely to succeed (the Scheduler role).

3. **Generate a CoT candidate answer.** Reason through the question in natural language step by step, referencing specific cells and values from the table. Produce a concise final answer. Run this exactly once (no iterative refinement for CoT to save cost).

4. **Generate a PoT candidate answer.** Write Python code (using pandas) that loads the table, performs the required operations, and prints the answer. Execute the code. If it errors, debug by examining the traceback, fix the code, and re-execute (up to 3 iterations). Stop early if two consecutive iterations produce the same result.

5. **Generate a t2SA candidate answer.** Convert the table to a SQL-compatible schema (CREATE TABLE + INSERT statements or use sqlite3 with a DataFrame). Write a SQL query that answers the question. Execute it. If it errors, debug and retry (up to 3 iterations with early stopping).

6. **Apply the Scheduler optimization.** If the question type suggests one code path is clearly better (e.g., aggregation with GROUP BY favors SQL; complex string manipulation favors Python), run that path first. If its answer matches CoT, skip the other code path entirely.

7. **Score candidate answers with the Confidence Checker.** Compare each candidate answer against the question and table context. Assign a confidence score. If the answers agree across paths, confidence is high. If they diverge, flag the disagreement.

8. **Select or arbitrate the final answer.** If any candidate exceeds the confidence threshold, select the highest-scoring answer directly. If all candidates score low (meaning high uncertainty), invoke a Judge step: present all candidates with their reasoning traces and let a fresh reasoning pass select the best one.

9. **Format the final answer.** If the selected answer is verbose (>100 characters), extract the concise entity or value. Normalize formatting (numbers, dates, units) to match the expected output style.

10. **Return the answer with a confidence indicator and reasoning trace.** Show which path(s) produced the answer, whether they agreed, and the confidence level. This transparency lets the user verify the result.

## Concrete Examples

**Example 1: Numerical aggregation over a sales table**

User: "Here's our Q3 sales data. What was the total revenue for the North region?"

```
| Region | Month | Revenue  | Units |
|--------|-------|----------|-------|
| North  | Jul   | 145,000  | 290   |
| South  | Jul   | 132,000  | 264   |
| North  | Aug   | 158,000  | 316   |
| South  | Aug   | 141,000  | 282   |
| North  | Sep   | 163,000  | 326   |
| South  | Sep   | 155,000  | 310   |
```

Approach:
1. Parse table into DataFrame with proper numeric types (strip commas from Revenue).
2. **CoT path**: "North region rows are Jul=145000, Aug=158000, Sep=163000. Sum = 145000+158000+163000 = 466,000." Answer: 466,000.
3. **PoT path**: `df[df['Region']=='North']['Revenue'].sum()` -> 466000. Answer: 466,000.
4. **t2SA path**: `SELECT SUM(Revenue) FROM sales WHERE Region = 'North'` -> 466000. Answer: 466,000.
5. All three paths agree. Confidence: high. Skip Judge.

Output:
```
The total revenue for the North region in Q3 was $466,000.

Reasoning: All three analysis paths (text reasoning, Python, SQL) independently
produced the same result. Confidence: high.
```

**Example 2: Comparative question requiring multi-step reasoning**

User: "Which player had the highest improvement in score between Round 1 and Round 3?"

```
| Player   | Round 1 | Round 2 | Round 3 |
|----------|---------|---------|---------|
| Alice    | 72      | 68      | 65      |
| Bob      | 80      | 75      | 71      |
| Carol    | 75      | 74      | 70      |
| David    | 78      | 70      | 69      |
```

Approach:
1. Parse table. Note: in golf, lower is better, so "improvement" = Round 1 - Round 3.
2. **CoT path**: Alice: 72-65=7, Bob: 80-71=9, Carol: 75-70=5, David: 78-69=9. Bob and David tie at 9. Answer: "Bob and David (both improved by 9)."
3. **PoT path**:
   ```python
   df['improvement'] = df['Round 1'] - df['Round 3']
   best = df[df['improvement'] == df['improvement'].max()]
   print(best[['Player','improvement']].to_string(index=False))
   ```
   Output: Bob 9, David 9. Answer: "Bob and David."
4. **t2SA path**: `SELECT Player, (Round_1 - Round_3) AS improvement FROM scores ORDER BY improvement DESC LIMIT 2` -> Bob 9, David 9.
5. All paths agree on the tie. Confidence: high.

Output:
```
Bob and David both had the highest improvement of 9 strokes between Round 1
and Round 3.
```

**Example 3: Ambiguous question where paths disagree**

User: "What percentage of employees are in Engineering?" (table has 47 rows, some departments labeled "Engineering", others "Engineering - Platform", "Engineering - ML")

Approach:
1. **CoT path**: Counts only exact "Engineering" matches -> 8/47 = 17.0%.
2. **PoT path**: `df[df['Department']=='Engineering'].shape[0] / len(df) * 100` -> 17.0%.
3. **t2SA path**: `SELECT COUNT(*) * 100.0 / (SELECT COUNT(*) FROM emp) FROM emp WHERE Department LIKE 'Engineering%'` -> 38.3%.
4. Paths disagree (17.0% vs 38.3%). Confidence Checker scores are low for all.
5. **Judge step invoked**: Reviews the table, notes sub-departments. Determines the question is ambiguous. Reports both interpretations.

Output:
```
This depends on how "Engineering" is defined:
- Exact match ("Engineering" only): 8 of 47 employees (17.0%)
- All engineering departments (Engineering, Engineering - Platform,
  Engineering - ML): 18 of 47 employees (38.3%)

The broader interpretation (38.3%) is more likely what's intended, but the
question is ambiguous. Paths disagreed, which flagged this for closer review.
```

## Best Practices

- **Do:** Run CoT first and only once — it's cheap and provides a baseline for comparison. Use code paths for verification, not the other way around.
- **Do:** Normalize table data before reasoning (strip currency symbols, parse dates, handle nulls). Garbage in produces disagreement across all paths.
- **Do:** Compare answers across paths before returning. Agreement across independent methods is the strongest signal of correctness.
- **Do:** Use the Scheduler heuristic — if the question is clearly numerical aggregation, prioritize Python/SQL and skip the other if it agrees with CoT. This cuts LLM calls by 10-15%.
- **Avoid:** Running all three paths on trivial lookups (e.g., "What is the value in row 2, column 3?"). CoT alone suffices for direct cell references.
- **Avoid:** Trusting a single code execution without checking for silent errors — a Python script that runs without exceptions can still return wrong results from type coercion or off-by-one filtering.
- **Avoid:** Over-relying on the Judge for every disagreement. First check whether the disagreement stems from formatting differences (e.g., "466000" vs "$466,000") rather than substantive errors.

## Error Handling

| Problem | Detection | Resolution |
|---------|-----------|------------|
| Code execution fails | Python/SQL raises an exception | Feed the full traceback to a debug prompt. Retry up to 3 times with corrected code. |
| Table too large for context | Token count exceeds model limit | Truncate to relevant columns/rows based on question keywords. Use `adjust_context` to fit within limits. |
| All paths disagree | No two candidates match | Invoke the Judge with all candidates and their reasoning traces. If Judge is also uncertain, return all candidates with caveats. |
| Formatting mismatch | Answers match semantically but not literally (e.g., "3" vs "three") | Normalize answers before comparison: convert words to numbers, strip units, standardize date formats. |
| Ambiguous question | Confidence scores low across all paths | Surface the ambiguity to the user. Present interpretations rather than guessing. |
| Mixed data types in column | Code path throws type error | Cast columns explicitly based on majority type. Handle exceptions per-cell with coercion or NaN. |

## Limitations

- **Context window constraints**: Very large tables (hundreds of columns or thousands of rows) cannot fit in a single prompt. You must pre-filter relevant columns and sample rows, which risks missing data.
- **Domain-specific tables**: Tables with specialized notation (chemical formulas, musical keys, encoded IDs) may confuse all three reasoning paths equally. Domain knowledge is not a substitute for reasoning diversity.
- **Subjective questions**: MATA is designed for factual, verifiable answers. Questions like "Which product is best?" have no ground truth for confidence scoring to work against.
- **SQL schema inference**: Automatically generating correct SQL schemas from messy tables (inconsistent headers, nested values) is error-prone. Manual schema hints improve reliability.
- **Overhead for simple questions**: The multi-path approach adds latency. For trivial lookups, a single CoT pass is faster and equally accurate — use the Scheduler logic to avoid unnecessary work.

## Reference

**Paper**: [MATA: Multi-Agent Framework for Reliable and Flexible Table Question Answering](https://arxiv.org/abs/2602.09642v1) (Hyeon et al., 2026). Look for Algorithm 1 (Scheduler-based agent selection), Algorithm 3 (confidence-threshold answer selection), and the ablation study showing the Confidence Checker is the single most impactful component. Code: [github.com/AIDAS-Lab/MATA](https://github.com/AIDAS-Lab/MATA).