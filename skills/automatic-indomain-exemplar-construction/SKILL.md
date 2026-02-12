---
name: "automatic-indomain-exemplar-construction"
description: "Query expansion with large language models is promising but often relies on hand-crafted prompts, manually chosen exemplars, or a single LLM, making it non-scalable and sensitive to domain shift. Implements techniques from the paper 'Automatic In-Domain Exemplar Construction and LLM-Based Refinement of Multi-LLM Expansions for Query Expansion' for optimize prompts for better ai model performance. Use when tasks involve (prompt engineering), (security) or when the user references techniques from this research area."
---

# Automatic In-Domain Exemplar Construction and LLM-Based Refinement of Multi-LLM Expansions for Query Expansion

**Source:** [https://arxiv.org/abs/2602.08917v1](https://arxiv.org/abs/2602.08917v1)
**Category:** cs.IR | **Published:** 2026-02-09 | **Skill Score:** 58
**Authors:** Minghan Li, Ercong Nie, Siqi Zhao...

## Core Capability

Optimize prompts for better AI model performance.

## Key Techniques

- **Proposed technique:** an automated

## Workflow

1. Analyze the current prompt and its shortcomings
2. Apply prompt engineering techniques (CoT, few-shot, etc.)
3. Test prompts against diverse inputs
4. Iterate on prompt design based on results
5. Document the prompt template and its parameters

## Security Considerations

- Check for OWASP Top 10 vulnerabilities
- Validate all user inputs at system boundaries
- Use parameterized queries for database access
- Follow the principle of least privilege

## Research Context

> Query expansion with large language models is promising but often relies on hand-crafted prompts, manually chosen exemplars, or a single LLM, making it non-scalable and sensitive to domain shift. We present an automated, domain-adaptive QE framework that builds in-domain exemplar pools by harvesting pseudo-relevant passages using a BM25-MonoT5 pipeline. A training-free cluster-based strategy selects diverse demonstrations, yielding strong and stable in-context QE without supervision. To further 

Refer to the [full paper](https://arxiv.org/abs/2602.08917v1) for detailed methodology.