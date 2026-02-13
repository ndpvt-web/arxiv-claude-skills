---
name: "birdturk-adaptation-bird-text-to-sql"
description: "Adapt Text-to-SQL systems and benchmarks for non-English, morphologically rich languages using controlled translation pipelines and agentic multi-stage reasoning. Triggers: 'translate SQL benchmark to Turkish', 'cross-lingual text-to-SQL', 'adapt BIRD dataset to another language', 'build multilingual SQL generation', 'text-to-SQL for low-resource language', 'non-English database querying'"
---

# Cross-Lingual Text-to-SQL Adaptation with Agentic Reasoning

This skill enables Claude to adapt English Text-to-SQL benchmarks and systems to morphologically rich, low-resource languages using the BIRDTurk methodology. The core technique is a controlled translation pipeline that adapts natural language questions, schema identifiers, database values, and hints to a target language while strictly preserving SQL logical structure and execution semantics. It also applies agentic multi-stage reasoning, which the paper demonstrates is the most robust approach for cross-lingual SQL generation, outperforming both direct prompting and standard fine-tuning.

## When to Use

- When the user needs to build a Text-to-SQL system that works in a non-English language (Turkish, Arabic, Korean, Finnish, or any agglutinative/morphologically rich language)
- When adapting an existing English SQL benchmark (BIRD, Spider, WikiSQL) to another language for evaluation
- When the user wants to generate SQL from natural language questions written in a low-resource language
- When evaluating how well LLMs handle structured reasoning in non-English contexts
- When the user needs to translate database schemas (table names, column names, value descriptions) while keeping SQL queries functionally identical
- When building multi-stage agentic pipelines for cross-lingual database querying
- When fine-tuning models for multilingual Text-to-SQL and encountering performance degradation

## Key Technique

BIRDTurk introduces a **controlled translation pipeline** with a critical constraint: SQL queries and database execution semantics must remain identical before and after translation. This means you translate the natural language question, the schema identifiers (table names, column names), database cell values, and evidence hints, but the SQL query structure stays unchanged. The mapping between original and translated identifiers is maintained as a bijective function so every translated column name maps back to exactly one original column. Translation quality is validated statistically using Central Limit Theorem sampling (n >= 30 per stratum) to achieve 95% confidence, reaching 98.15% accuracy on human evaluation.

The paper evaluates three approaches for generating SQL from translated inputs: (1) **inference-based prompting** (zero-shot and few-shot with schema context), (2) **agentic multi-stage reasoning** (decomposing the problem into schema understanding, intermediate SQL generation, execution-based validation, and iterative refinement), and (3) **supervised fine-tuning** on translated training data. The key finding is that agentic multi-stage reasoning demonstrates the strongest cross-lingual robustness. By breaking the problem into discrete stages — schema analysis, query planning, SQL generation, execution feedback — each stage can compensate for linguistic ambiguity introduced by morphological complexity. Direct prompting suffers the largest performance drop because the model must handle linguistic parsing and SQL generation in a single pass.

Supervised fine-tuning on standard multilingual baselines (e.g., mBERT-family models) struggles with Turkish due to underrepresentation in pretraining data. However, modern instruction-tuned models (GPT-4, Qwen2) scale effectively with fine-tuning, showing that the base model's multilingual capacity is a prerequisite. The consistent finding across all approaches: Turkish introduces 10-25% execution accuracy degradation compared to English, driven by agglutinative morphology, free word order, and vowel harmony creating token distributions unseen during pretraining.

## Step-by-Step Workflow

### For Adapting a Benchmark to a New Language

1. **Inventory the source benchmark structure.** Catalog all translatable components: natural language questions, schema identifiers (table names, column names), column descriptions, database cell values containing natural language text, and evidence/hint fields. Do NOT translate SQL keywords, operators, or structural tokens.

2. **Build a schema translation dictionary.** For each database in the benchmark, create a bijective mapping from English identifiers to target-language identifiers. Ensure every translated name is a valid SQL identifier (no spaces unless quoted, no reserved words). For agglutinative languages, prefer root forms over inflected forms for column names.

3. **Translate natural language questions with schema grounding.** Translate each question while ensuring that references to schema elements use the exact translated identifier from your dictionary. Preserve the question's logical intent — the same SQL query must answer both the English and translated question.

4. **Translate database cell values selectively.** Only translate cell values that contain natural language text (e.g., city names, descriptions). Leave numeric values, codes, and IDs untouched. Update any `WHERE` clause string literals in the SQL to match the translated cell values.

5. **Translate evidence hints and descriptions.** These provide domain context for SQL generation. Translate them while preserving technical accuracy — a hint about date formats or business rules must remain semantically identical.

6. **Validate translation quality statistically.** Sample at least 30 translated question-SQL pairs per domain stratum. Have human evaluators check: (a) the translated question is natural in the target language, (b) the SQL query still correctly answers the translated question against the translated database, (c) execution results are identical.

7. **Run execution-based validation.** Execute every SQL query against the translated database and compare results to the original. Any execution mismatch indicates a translation error in cell values or schema identifiers.

### For Building a Cross-Lingual Text-to-SQL System

8. **Implement agentic multi-stage reasoning.** Structure the pipeline as: Stage 1 (Schema Analysis) — parse the translated schema, identify relevant tables/columns, resolve morphological variants; Stage 2 (Query Planning) — decompose the natural language question into sub-goals mapped to SQL operations; Stage 3 (SQL Generation) — produce candidate SQL; Stage 4 (Execution Validation) — run the SQL, check for errors or empty results, refine if needed.

9. **Design prompts with explicit schema context.** Include the full translated schema (table names, column names, types, sample values) in the prompt. For morphologically rich languages, also include the original English identifier as a disambiguation aid in parentheses.

10. **Evaluate with execution accuracy (EX).** The primary metric is whether the generated SQL produces the same result set as the gold SQL when executed against the database. Do not rely on exact-match string comparison of SQL, as valid queries can have many equivalent forms.

## Concrete Examples

**Example 1: Adapting a database schema from English to Turkish**

User: "I have a BIRD-format database with tables `employees(id, name, department, salary)` and `departments(id, dept_name, location)`. Adapt this schema for Turkish Text-to-SQL evaluation."

Approach:
1. Build the schema translation dictionary
2. Translate table and column names to natural Turkish equivalents
3. Update any natural-language cell values
4. Keep SQL structure intact

Output:
```sql
-- Schema Translation Dictionary
-- employees -> calisanlar
-- id -> id  (universal, no translation needed)
-- name -> isim
-- department -> departman
-- salary -> maas
-- departments -> departmanlar
-- dept_name -> departman_adi
-- location -> konum

-- Original English question:
-- "What is the average salary of employees in the Sales department?"

-- Translated Turkish question:
-- "Satis departmanindaki calisanlarin ortalama maasi nedir?"

-- SQL (unchanged structure, translated identifiers):
SELECT AVG(maas) FROM calisanlar
WHERE departman = 'Satis';

-- Note: 'Sales' cell value translated to 'Satis' in the database
-- and in the WHERE clause literal
```

**Example 2: Agentic multi-stage pipeline for Turkish Text-to-SQL**

User: "Build a pipeline that takes a Turkish natural language question and generates SQL against a translated schema."

Approach:
1. Implement a 4-stage agentic pipeline
2. Each stage has a focused prompt and validation

Output:
```python
import openai

def text_to_sql_agentic(question_tr: str, schema: dict, db_connection) -> str:
    """Multi-stage agentic Text-to-SQL for Turkish."""

    # Stage 1: Schema Analysis
    schema_prompt = f"""Given this database schema:
{format_schema(schema)}

Turkish question: {question_tr}

Identify which tables and columns are relevant to answering
this question. For each column, note its data type and any
value constraints. Turkish morphology note: the question may
use agglutinative suffixes (-daki, -nin, -larin) that modify
base words matching column names."""

    relevant_schema = llm_call(schema_prompt)

    # Stage 2: Query Planning
    plan_prompt = f"""Relevant schema elements:
{relevant_schema}

Turkish question: {question_tr}

Decompose this question into SQL operations:
- What aggregation is needed? (COUNT, AVG, SUM, etc.)
- What filtering conditions apply?
- Are JOINs required? Between which tables?
- What is the expected result shape?"""

    query_plan = llm_call(plan_prompt)

    # Stage 3: SQL Generation
    sql_prompt = f"""Based on this query plan:
{query_plan}

Full schema: {format_schema(schema)}
Generate the SQL query. Use exact column and table names
from the schema. Do not invent column names."""

    candidate_sql = llm_call(sql_prompt)

    # Stage 4: Execution Validation
    try:
        result = db_connection.execute(candidate_sql)
        if result.rowcount == 0:
            # Refine: empty result may indicate wrong filter
            refined_sql = refine_with_feedback(
                candidate_sql, "Query returned 0 rows", schema, question_tr
            )
            return refined_sql
        return candidate_sql
    except Exception as e:
        # Refine: execution error indicates structural problem
        refined_sql = refine_with_feedback(
            candidate_sql, str(e), schema, question_tr
        )
        return refined_sql
```

**Example 3: Validating translation quality with statistical sampling**

User: "I translated 12,000 question-SQL pairs to Korean. How do I validate the translation quality?"

Approach:
1. Apply Central Limit Theorem-based sampling
2. Stratify by database domain
3. Define evaluation criteria

Output:
```python
import math
import random

def compute_sample_size(population_size: int, confidence_z: float = 1.96,
                        margin_error: float = 0.05, estimated_p: float = 0.5) -> int:
    """Compute required sample size for translation validation.
    BIRDTurk uses 95% confidence (z=1.96), 5% margin of error."""
    n0 = (confidence_z ** 2 * estimated_p * (1 - estimated_p)) / (margin_error ** 2)
    # Finite population correction
    n = n0 / (1 + (n0 - 1) / population_size)
    return math.ceil(n)

# For 12,000 pairs: sample size ~= 373
sample_size = compute_sample_size(12000)

# Stratify by domain to ensure coverage
def stratified_sample(pairs_by_domain: dict, total_sample: int) -> list:
    """Sample proportionally from each domain."""
    total = sum(len(v) for v in pairs_by_domain.values())
    sample = []
    for domain, pairs in pairs_by_domain.items():
        k = max(1, round(total_sample * len(pairs) / total))
        sample.extend(random.sample(pairs, min(k, len(pairs))))
    return sample

# Evaluation criteria per sample:
# 1. Is the translated question natural and grammatical? (Y/N)
# 2. Does the SQL still answer the translated question? (Y/N)
# 3. Does execution produce identical results? (Y/N)
# Target: >= 95% accuracy across all three criteria
```

## Best Practices

**Do:**
- Maintain a strict bijective mapping between original and translated schema identifiers. Every column name must map to exactly one translation and back.
- Use root/lemma forms for translated column names in agglutinative languages. Avoid inflected forms that could confuse the model.
- Include both the translated and original column names in prompts (e.g., `maas (salary)`) to help the model bridge linguistic gaps during early development.
- Validate every translated SQL query by execution against the translated database. String-matching SQL is insufficient since equivalent queries can differ syntactically.
- Test with at least 3 model families (e.g., GPT-4, Qwen2, Llama) because cross-lingual performance varies dramatically by pretraining data composition.

**Avoid:**
- Do not translate SQL keywords, operators, or syntax. `SELECT`, `WHERE`, `JOIN` remain in English regardless of target language.
- Do not translate numeric values, codes, or IDs in database cells. Only translate natural language content.
- Do not assume fine-tuning on a small translated dataset will close the performance gap. The paper shows standard multilingual baselines fail without sufficient base model multilingual capacity.
- Do not evaluate with exact SQL string matching. Use execution accuracy (EX) as the primary metric — whether the result sets are identical.

## Error Handling

- **Schema identifier collision:** Two English column names may translate to the same target-language word. Resolve by appending the table name as a disambiguator (e.g., `departman_adi` vs `proje_adi` instead of both becoming `ad`).
- **Cell value mismatch in WHERE clauses:** If you translate a cell value in the database but forget to update the corresponding SQL string literal, the query will return empty results. Always propagate value translations to all SQL queries referencing those values.
- **Morphological ambiguity:** Agglutinative suffixes can make the same base word appear as many surface forms. In the schema analysis stage of the agentic pipeline, explicitly strip common suffixes to match against column names.
- **Execution errors after translation:** If a translated SQL query fails, check for: (a) untranslated identifiers still referencing the English schema, (b) translated names that are SQL reserved words, (c) encoding issues with non-ASCII characters in identifiers.
- **Empty results on valid queries:** The agentic pipeline's Stage 4 catches this. Common cause: a `WHERE` filter uses the English cell value against a translated database. The refinement loop should detect and correct the language mismatch.

## Limitations

- The approach assumes a one-to-one mapping between source and target schema identifiers, which breaks down when languages lack direct equivalents for domain-specific terms (e.g., financial jargon).
- Performance degradation of 10-25% on Turkish compared to English is structural and cannot be fully eliminated by pipeline engineering alone — it reflects gaps in LLM pretraining data.
- The controlled translation pipeline requires human validation, making it labor-intensive. Fully automated translation (e.g., using machine translation APIs) risks semantic drift that corrupts SQL execution semantics.
- Languages with non-Latin scripts (Chinese, Arabic, Thai) introduce additional tokenization challenges not fully addressed by this methodology.
- Fine-tuning results depend heavily on the base model's existing multilingual capacity. If a language is severely underrepresented in pretraining, no amount of fine-tuning on a few thousand examples will achieve parity with English.
- The benchmark only evaluates single-turn Text-to-SQL. Multi-turn conversational SQL generation across languages is an open problem.

## Reference

**Paper:** [BIRDTurk: Adaptation of the BIRD Text-to-SQL Dataset to Turkish](https://arxiv.org/abs/2602.03633v1) (EACL 2026 SIGTURK)
**Dataset:** [metunlp/birdturk on HuggingFace](https://huggingface.co/datasets/metunlp/birdturk)
**Key takeaway:** Agentic multi-stage reasoning (schema analysis → query planning → SQL generation → execution validation) provides the strongest cross-lingual robustness for Text-to-SQL, outperforming both direct prompting and fine-tuning on morphologically rich, low-resource languages.