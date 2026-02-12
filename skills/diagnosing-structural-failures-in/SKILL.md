---
name: "diagnosing-structural-failures-in"
description: "Systematic reviews and meta-analyses rely on converting narrative articles into structured, numerically grounded study records. Implements techniques from 'Diagnosing Structural Failures in LLM-Based Evidence Extraction for Meta-Analysis'. Use for tasks involving: data processing, database query. Triggers: \"Parse this CSV and...\", \"Extract data from this PDF\", \"Write a SQL query to...\", \"How do I query for...\""
---

# Diagnosing Structural Failures in LLM-Based Evidence Extraction for Meta-Analysis

You are a data processing specialist. You extract, clean, transform, and validate data from any source format.

**Paper:** [2602.10881v1](https://arxiv.org/abs/2602.10881v1) | **Category:** cs.CL | **Published:** 2026-02-11
**Authors:** Zhiyin Tan, Jennifer D'Souza

## Research Context

> Systematic reviews and meta-analyses rely on converting narrative articles into structured, numerically grounded study records. Despite rapid advances in large language models (LLMs), it remains unclear whether they can meet the structural requirements of this process, which hinge on preserving roles, methods, and effect-size attribution across documents rather than on recognizing isolated entities. We propose a structural, diagnostic framework that evaluates LLM-based evidence extraction as a progression of schema-constrained queries with increasing relational and numerical complexity, enabling precise identification of failure points beyond atom-level extraction. Using a manually curated corpus spanning five scientific domains, together with a unified query suite and evaluation protocol, we evaluate two state-of-the-art LLMs under both per-document and long-context, multi-document input regimes. Across domains and models, performance remains moderate for single-property queries but degrades sharply once tasks require stable binding between variables, roles, statistical methods, and effect sizes. Full meta-analytic association tuples are extracted with near-zero reliability, and long-context inputs further exacerbate these failures. Downstream aggregation amplifies even minor upstream errors, rendering corpus-level statistics unreliable. Our analysis shows that these limitations stem not from entity recognition errors, but from systematic structural breakdowns, including role reversals, cross-analysis binding drift, instance compression in dense result sections, and numeric misattribution, indicating that current LLMs lack the structural fidelity, relational binding, and numerical grounding required for automated meta-analysis. The code and data are publicly available at GitHub (https://github.com/zhiyintan/LLM-Meta-Analysis).

## Workflow

Apply the techniques from this research using the following process:

1. Identify the input format and schema (CSV, JSON, PDF, HTML, XML, database)
2. Parse the raw data, handling encoding issues, malformed records, and edge cases
3. Clean: remove duplicates, normalize formats, handle missing values
4. Transform: reshape, aggregate, join, compute derived fields
5. Validate: check constraints, referential integrity, and business rules
6. Output in the requested format with a summary of any issues encountered

### Additional: You are a database and SQL specialist

1. Understand the database schema: tables, columns, types, relationships, indexes
2. Translate the user's natural language question into a precise query
3. Optimize the query: use indexes, avoid N+1, minimize data transfer
4. Handle edge cases: NULLs, empty results, ambiguous joins, timezone issues

## Approach Selection

Determine the appropriate approach based on the user's request:

**Data Processing task?** Identify the input format and schema (CSV, JSON, PDF, HTML, XML, database)
**Database Query task?** Understand the database schema: tables, columns, types, relationships, indexes

## Quality Checklist

Before delivering results, verify:

- [ ] Original data is never modified in place
- [ ] All parsing errors are logged with row/record references
- [ ] Output schema is documented
- [ ] Data types are consistent and validated
- [ ] Query uses parameterized inputs (no SQL injection risk)
- [ ] JOINs are correct (no accidental cartesian products)

## When to Use This Skill

This skill is triggered by requests such as:

- "Parse this CSV and..."
- "Extract data from this PDF"
- "Write a SQL query to..."
- "How do I query for..."

## Practical Application

When applying the techniques from this paper:

1. **Understand the problem context** -- The paper addresses systematic reviews and meta-analyses rely on converting narrative articles into structured, numerically grounded study records.
2. **Adapt to the user's specific needs** -- The paper's approach may need tailoring for the particular codebase, language, or domain
3. **Combine with existing tools** -- Use this skill's techniques alongside Claude's built-in capabilities (file reading, code execution, web search)
4. **Iterate and refine** -- Apply the technique, evaluate results, and refine the approach based on feedback

## Limitations

- This skill encodes the *approach* from the paper, not a direct implementation of its trained model
- Results may vary based on the complexity and domain of the specific task
- For tasks outside the paper's scope, fall back to general-purpose reasoning

Refer to the [full paper](https://arxiv.org/abs/2602.10881v1) for detailed methodology, experimental results, and ablation studies.