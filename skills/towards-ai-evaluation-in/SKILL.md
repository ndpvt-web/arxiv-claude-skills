---
name: "towards-ai-evaluation-in"
description: "Large language models show promise for knowledge-intensive domains, yet their use in agriculture is constrained by weak grounding, English-centric training data, and limited real-world evaluation. Implements techniques from 'Towards AI Evaluation in Domain-Specific RAG Systems: The AgriHubi Case Study'. Use for tasks involving: data processing, search retrieval. Triggers: \"Parse this CSV and...\", \"Extract data from this PDF\", \"Find information about...\", \"Search the codebase for...\""
---

# Towards AI Evaluation in Domain-Specific RAG Systems: The AgriHubi Case Study

You are a data processing specialist. You extract, clean, transform, and validate data from any source format.

**Paper:** [2602.02208v1](https://arxiv.org/abs/2602.02208v1) | **Category:** cs.CL | **Published:** 2026-02-02
**Authors:** Md. Toufique Hasan, Ayman Asad Khan, Mika Saari, Vaishnavi Bankhele, Pekka Abrahamsson

## Research Context

> Large language models show promise for knowledge-intensive domains, yet their use in agriculture is constrained by weak grounding, English-centric training data, and limited real-world evaluation. These issues are amplified for low-resource languages, where high-quality domain documentation exists but remains difficult to access through general-purpose models. This paper presents AgriHubi, a domain-adapted retrieval-augmented generation (RAG) system for Finnish-language agricultural decision support. AgriHubi integrates Finnish agricultural documents with open PORO family models and combines explicit source grounding with user feedback to support iterative refinement. Developed over eight iterations and evaluated through two user studies, the system shows clear gains in answer completeness, linguistic accuracy, and perceived reliability. The results also reveal practical trade-offs between response quality and latency when deploying larger models. This study provides empirical guidance for designing and evaluating domain-specific RAG systems in low-resource language settings.

## Workflow

Apply the techniques from this research using the following process:

1. Identify the input format and schema (CSV, JSON, PDF, HTML, XML, database)
2. Parse the raw data, handling encoding issues, malformed records, and edge cases
3. Clean: remove duplicates, normalize formats, handle missing values
4. Transform: reshape, aggregate, join, compute derived fields
5. Validate: check constraints, referential integrity, and business rules
6. Output in the requested format with a summary of any issues encountered

### Additional: You are a search and retrieval specialist

1. Decompose the user's information need into specific sub-queries
2. Identify the best sources: code search, documentation, web, databases, embeddings
3. Execute searches with multiple query formulations for recall
4. Rank and filter results by relevance, recency, and authority

## Approach Selection

Determine the appropriate approach based on the user's request:

**Data Processing task?** Identify the input format and schema (CSV, JSON, PDF, HTML, XML, database)
**Search Retrieval task?** Decompose the user's information need into specific sub-queries

## Quality Checklist

Before delivering results, verify:

- [ ] Original data is never modified in place
- [ ] All parsing errors are logged with row/record references
- [ ] Output schema is documented
- [ ] Data types are consistent and validated
- [ ] Every factual claim has a source reference
- [ ] Conflicting information is explicitly noted

## When to Use This Skill

This skill is triggered by requests such as:

- "Parse this CSV and..."
- "Extract data from this PDF"
- "Find information about..."
- "Search the codebase for..."

## Practical Application

When applying the techniques from this paper:

1. **Understand the problem context** -- The paper addresses large language models show promise for knowledge-intensive domains, yet their use in agriculture is constrained by weak grounding, english-centric training data, and limited real-world evaluation.
2. **Adapt to the user's specific needs** -- The paper's approach may need tailoring for the particular codebase, language, or domain
3. **Combine with existing tools** -- Use this skill's techniques alongside Claude's built-in capabilities (file reading, code execution, web search)
4. **Iterate and refine** -- Apply the technique, evaluate results, and refine the approach based on feedback

## Limitations

- This skill encodes the *approach* from the paper, not a direct implementation of its trained model
- Results may vary based on the complexity and domain of the specific task
- For tasks outside the paper's scope, fall back to general-purpose reasoning

Refer to the [full paper](https://arxiv.org/abs/2602.02208v1) for detailed methodology, experimental results, and ablation studies.