---
name: "ghostcite-a-largescale-analysis"
description: "Citations provide the basis for trusting scientific claims; when they are invalid or fabricated, this trust collapses. Implements techniques from the paper 'GhostCite: A Large-Scale Analysis of Citation Validity in the Age of Large Language Models' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (security) or when the user references techniques from this research area."
---

# GhostCite: A Large-Scale Analysis of Citation Validity in the Age of Large Language Models

**Source:** [https://arxiv.org/abs/2602.06718v1](https://arxiv.org/abs/2602.06718v1)
**Category:** cs.CR | **Published:** 2026-02-06 | **Skill Score:** 62
**Authors:** Zuyao Xu, Yuqi Qiu, Lu Sun...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** citeverifier

## Workflow

1. Parse the user's information need or query
2. Formulate effective search strategies
3. Retrieve relevant documents, code, or data
4. Synthesize findings into a coherent response
5. Provide citations and references

## Security Considerations

- Check for OWASP Top 10 vulnerabilities
- Validate all user inputs at system boundaries
- Use parameterized queries for database access
- Follow the principle of least privilege

## Research Context

> Citations provide the basis for trusting scientific claims; when they are invalid or fabricated, this trust collapses. With the advent of Large Language Models (LLMs), this risk has intensified: LLMs are increasingly used for academic writing, yet their tendency to fabricate citations (``ghost citations'') poses a systemic threat to citation validity.   To quantify this threat and inform mitigation, we develop CiteVerifier, an open-source framework for large-scale citation verification, and cond

Refer to the [full paper](https://arxiv.org/abs/2602.06718v1) for detailed methodology.