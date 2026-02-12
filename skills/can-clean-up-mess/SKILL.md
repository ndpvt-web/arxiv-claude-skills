---
name: "can-clean-up-mess"
description: "LLM-driven data preparation pipeline for cleaning, integrating, and enriching messy datasets. Use when the user says 'clean this data', 'fix this CSV', 'match these schemas', 'deduplicate these records', 'impute missing values', or 'annotate this table'."
---

# LLM-Driven Data Preparation: Clean, Integrate, and Enrich Messy Data

This skill enables Claude to act as an agentic data preparation system that applies prompt-driven techniques from the survey "Can LLMs Clean Up Your Mess?" (arXiv:2601.17058). Instead of writing bespoke rule-based cleaning scripts, Claude uses structured serialization, few-shot prompting, chain-of-thought reasoning, and iterative detect-verify-repair loops to standardize formats, detect and correct errors, impute missing values, match entities across datasets, align schemas, and annotate columns with semantic types. The approach replaces fragile regex/rule pipelines with context-aware LLM reasoning grounded in the actual data.

## When to Use

- When the user has a CSV, JSON, or database table with inconsistent formats (mixed date formats, inconsistent casing, varying unit representations) and asks to standardize it
- When the user wants to detect and fix data errors such as typos, out-of-range values, constraint violations, or duplicate entries in a dataset
- When the user has missing values in a table and needs intelligent imputation that respects column correlations and domain semantics
- When the user needs to match records across two datasets that refer to the same real-world entities (entity resolution / deduplication)
- When the user has two schemas or tables and needs to align columns that represent the same concept but use different names
- When the user wants to automatically annotate columns with semantic types, expand abbreviated headers, or profile a dataset's structure
- When the user asks to build a reusable data cleaning or preparation pipeline for recurring ingestion tasks

## Key Technique

The paper identifies a paradigm shift from deterministic rule-based data preparation to **prompt-driven, context-aware, agentic workflows**. The core insight is that LLMs can understand data semantics — they recognize that "NYC", "New York City", and "new york" are the same entity, that a zip code constrains a city name, and that a column of dates in mixed formats should converge to ISO 8601 — without needing hand-coded rules for each case.

Three interlocking strategies make this practical. First, **structured serialization**: tabular data is converted into natural-language or semi-structured representations (row-as-sentence, column-sample lists, or JSON records) that fit within prompt context. Selective context — choosing only the columns most correlated with the target via Pearson, Cramer's V, or eta correlation — keeps prompts focused and cost-efficient. Second, **iterative detect-verify-repair loops**: rather than making a single pass, the LLM cycles through detection (flag suspicious values), self-verification (confirm the flagged values are truly errors given the full row context), and repair (generate corrected values), reducing hallucinated corrections. Third, **code synthesis over direct generation**: for repeatable transformations, the LLM generates executable cleaning functions (Python/SQL) that can be validated, tested, and reused, rather than producing corrected values one-by-one — this is both cheaper and auditable.

For integration tasks like entity matching, the paper highlights **batch-clustering prompts** (process groups of candidate pairs together so the LLM can reason about inter-pair relationships) and **retrieval-augmented matching** (retrieve similar resolved pairs as few-shot examples). For enrichment, **self-reflection annotation** — where the LLM annotates, then reviews its own annotations for consistency — improves label quality without human review.

## Step-by-Step Workflow

1. **Profile the data**: Load the dataset and generate a structural profile — column names, inferred types, cardinality, null rates, sample values (10-20 per column), and basic statistics. This is the foundation for every subsequent step.

2. **Serialize strategically**: Convert the relevant portion of data into a prompt-friendly format. For row-level tasks (error detection, imputation), serialize individual rows as key-value pairs with column context. For column-level tasks (standardization, annotation), serialize sampled values from the target column grouped as a list. Keep total token count under control by sampling representative values via clustering or stratified sampling.

3. **Detect issues with chain-of-thought prompting**: Ask the LLM to examine serialized data and identify problems — inconsistent formats, likely errors, missing patterns — using explicit step-by-step reasoning. Prompt the LLM to state what the expected format/range is before flagging deviations. This reduces false positives compared to zero-shot detection.

4. **Verify detections against row context**: For each flagged issue, re-prompt with the full row (and optionally neighboring rows or correlated columns) to confirm the detection. Use a "self-consistency" check: if the LLM confirms the error in 2 out of 3 independent verification prompts, proceed to repair.

5. **Generate repair as executable code when possible**: Instead of asking the LLM to output corrected values directly, ask it to generate a Python function or SQL expression that performs the transformation. For example: `def standardize_date(val): ...` or `UPDATE t SET city = 'New York' WHERE zip = '10001' AND city = 'NYC'`. Validate the generated code against the sample data before applying it to the full dataset.

6. **For imputation, select context columns by correlation**: Compute pairwise correlation between the column with missing values and all other columns. Include only the top-k most correlated columns in the imputation prompt, plus the target column. Serialize the k nearest complete rows (by similarity on the correlated columns) as few-shot context.

7. **For entity matching, use batch-clustering prompts**: Serialize candidate pairs in batches of 5-10 per prompt. Include a brief instruction specifying matching criteria and 2-3 labeled examples (one match, one non-match, one ambiguous). Ask the LLM to classify each pair and explain its reasoning, then cluster matched entities.

8. **For schema matching, provide both schemas with sample data**: Serialize both schemas side-by-side with 3-5 sample values per column. Prompt the LLM to propose column mappings with confidence scores and brief justifications. Cross-validate by asking it to generate a test query that joins the two tables on the proposed mapping.

9. **Validate all outputs against the source data**: After any transformation, run automated checks — row count preservation, type consistency, value range constraints, referential integrity. Surface any violations back to the LLM for a second repair pass.

10. **Emit the final pipeline as reproducible code**: Consolidate all generated cleaning functions, mappings, and transformations into a single executable script (Python or SQL) with comments explaining each step. This makes the pipeline auditable and re-runnable on future data batches.

## Concrete Examples

**Example 1: Standardize inconsistent date formats in a CSV**

User: "This CSV has dates in column `created_at` with mixed formats — some are MM/DD/YYYY, some are YYYY-MM-DD, some are 'Jan 5, 2024'. Standardize them all to ISO 8601."

Approach:
1. Read the CSV and sample 20 values from `created_at` to identify the format variants
2. Serialize the samples into a prompt: "Given these date values: ['01/15/2024', '2024-02-03', 'Jan 5, 2024', '3/7/24', ...], write a Python function `standardize_date(val: str) -> str` that converts any of these formats to 'YYYY-MM-DD'. Handle edge cases like two-digit years."
3. Generate the function, test it against all 20 samples, fix any failures
4. Apply the function across the full column using pandas

Output:
```python
import pandas as pd
from dateutil import parser

def standardize_date(val):
    if pd.isna(val) or str(val).strip() == "":
        return None
    try:
        return parser.parse(str(val)).strftime("%Y-%m-%d")
    except (ValueError, TypeError):
        return None  # flag for manual review

df["created_at"] = df["created_at"].apply(standardize_date)
failed = df[df["created_at"].isna()]
print(f"Standardized {len(df) - len(failed)} rows, {len(failed)} need review")
```

**Example 2: Impute missing city names using zip codes and state**

User: "My customer table has 15% missing values in the `city` column but `zip_code` and `state` are mostly complete. Can you fill in the missing cities?"

Approach:
1. Profile correlations: `zip_code` has near-perfect correlation with `city` — use it as primary context
2. Build a lookup from complete rows: group by `zip_code`, take the most frequent `city` for each
3. For zip codes not seen in complete rows, serialize 5 rows with the same state as few-shot context and prompt the LLM: "Given zip code 30301 in state GA, and these nearby examples: [{zip: 30302, city: 'Atlanta'}, {zip: 30303, city: 'Atlanta'}], what city does 30301 correspond to?"
4. Validate LLM responses against a zip-code reference if available

Output:
```python
# Step 1: Deterministic lookup from existing data
zip_city_map = (
    df.dropna(subset=["city"])
    .groupby("zip_code")["city"]
    .agg(lambda x: x.mode().iloc[0])
    .to_dict()
)

# Step 2: Fill from lookup
mask = df["city"].isna()
df.loc[mask, "city"] = df.loc[mask, "zip_code"].map(zip_city_map)

# Step 3: LLM fallback for remaining gaps
still_missing = df[df["city"].isna()]
print(f"Filled {mask.sum() - len(still_missing)} via lookup, "
      f"{len(still_missing)} need LLM imputation")
# ... LLM imputation loop for remaining rows ...
```

**Example 3: Entity matching across two product catalogs**

User: "I have two product CSVs from different suppliers. I need to find which rows in catalog A refer to the same product as rows in catalog B. They use different naming conventions."

Approach:
1. Profile both catalogs: identify shared columns (product name, brand, category, price)
2. Generate candidate pairs using blocking on brand + category to reduce the O(n*m) comparison space
3. Serialize candidate pairs in batches of 8, with 2 labeled examples:
   ```
   Pair 1: A={"name": "Sony WH-1000XM5 Wireless NC Headphones", "brand": "Sony", "price": 348}
           B={"name": "SONY WH1000XM5 Noise Cancelling", "brand": "SONY", "price": 349.99}
   → Match? (explain reasoning)
   ```
4. Aggregate LLM decisions, resolve transitive conflicts (if A=B and A=C, then B=C)
5. Output a mapping table with match confidence

Output:
```
| catalog_a_id | catalog_b_id | confidence | reasoning                              |
|-------------|-------------|------------|----------------------------------------|
| A-1042      | B-887       | 0.95       | Same model number WH-1000XM5, ~$1 diff |
| A-2210      | B-1103      | 0.82       | Same brand+category, similar specs      |
| A-3301      | B-2045      | 0.60       | Similar name but different capacity     |
```

## Best Practices

- **Do**: Always profile data before cleaning — understanding null rates, cardinality, and type distributions prevents wasted LLM calls on non-issues
- **Do**: Generate code (Python functions, SQL expressions) rather than corrected values directly — code is testable, auditable, and reusable across data batches
- **Do**: Use batch serialization (5-10 items per prompt) for matching and classification tasks to amortize prompt overhead and let the LLM reason about inter-item relationships
- **Do**: Include 2-3 few-shot examples with explanations when prompting for error detection or entity matching — this grounds the LLM's decisions in concrete patterns
- **Avoid**: Sending entire large datasets through the LLM row by row — use sampling, blocking, and deterministic pre-filters to minimize LLM calls
- **Avoid**: Trusting LLM outputs without validation — always run automated checks (type constraints, row counts, value ranges, referential integrity) after any transformation
- **Avoid**: Using LLM imputation when a deterministic lookup (foreign key join, zip-to-city map) can resolve the missing value — LLMs are the fallback, not the first resort

## Error Handling

- **Hallucinated corrections**: The LLM may "fix" values that were actually correct. Mitigate by requiring the detect-verify-repair loop: flag, confirm with row context, then repair. If verification fails, leave the original value.
- **Inconsistent batch outputs**: When processing batches, the LLM may return different numbers of results than input rows. Always parse batch outputs with strict positional matching and re-prompt for any missing entries.
- **Code generation failures**: Generated cleaning functions may fail on edge cases not present in samples. Test against a diverse sample (including nulls, empty strings, unicode, extreme values) before applying to the full dataset. Wrap in try/except and route failures to a manual review queue.
- **Entity matching false positives**: Similar names can refer to different products (e.g., "iPhone 15" vs "iPhone 15 Pro"). When confidence is below 0.8, flag for human review rather than auto-merging.
- **Schema matching ambiguity**: Multiple plausible column mappings may exist (e.g., `created_date` could map to `date_added` or `registration_date`). Generate test join queries for each candidate mapping and compare result quality.

## Limitations

- **Scale**: Direct LLM-based cleaning is cost-prohibitive for datasets above ~100K rows without aggressive sampling and code-generation strategies. For large datasets, use the LLM to generate reusable transformation rules, then execute them deterministically.
- **Numerical reasoning**: LLMs are weak at precise numerical operations (statistical outlier detection, exact deduplication by numeric thresholds). Use traditional statistical methods for numeric anomaly detection and reserve LLMs for semantic tasks.
- **Domain-specific data**: Highly specialized data (genomic sequences, chemical formulas, engineering tolerances) may exceed the LLM's training distribution. For such domains, provide domain ontologies or reference tables in the prompt context.
- **Non-tabular data**: This workflow is optimized for structured and semi-structured tabular data. Unstructured text cleaning (e.g., deduplicating free-text paragraphs) requires different serialization strategies.
- **Determinism**: LLM outputs are non-deterministic. The same input may produce slightly different cleaning decisions across runs. For reproducibility, pin temperature to 0, log all prompts and responses, and validate outputs against fixed test cases.

## Reference

**Paper**: "Can LLMs Clean Up Your Mess? A Survey of Application-Ready Data Preparation with LLMs" — Zhou et al., arXiv:2601.17058 (2026). Read for: the full task taxonomy (cleaning/integration/enrichment), comparison of prompt-based vs. fine-tuned vs. agentic approaches per task, evaluation datasets and metrics, and the cost-quality tradeoff analysis. Repository: https://github.com/weAIDB/awesome-data-llm