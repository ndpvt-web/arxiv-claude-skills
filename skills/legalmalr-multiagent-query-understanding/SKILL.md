---
name: "legalmalr-multiagent-query-understanding"
description: "Statute retrieval is essential for legal assistance and judicial decision support, yet real-world legal queries are often implicit, multi-issue, and expressed in colloquial or underspecified forms. Implements techniques from the paper 'LegalMALR:Multi-Agent Query Understanding and LLM-Based Reranking for Chinese Statute Retrieval' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# LegalMALR:Multi-Agent Query Understanding and LLM-Based Reranking for Chinese Statute Retrieval

**Source:** [https://arxiv.org/abs/2601.17692v1](https://arxiv.org/abs/2601.17692v1)
**Category:** cs.IR | **Published:** 2026-01-25 | **Skill Score:** 74
**Authors:** Yunhan Li, Mingjie Xie, Gaoli Kang...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Retrieval-augmented** approach for grounding responses in external knowledge

## Workflow

1. Parse the user's information need or query
2. Formulate effective search strategies
3. Retrieve relevant documents, code, or data
4. Synthesize findings into a coherent response
5. Provide citations and references

## Agent Coordination

- Decompose complex tasks into independent subtasks where possible
- Use parallel execution for independent subtasks
- Implement error recovery and retry logic
- Aggregate partial results when full completion fails

## Research Context

> Statute retrieval is essential for legal assistance and judicial decision support, yet real-world legal queries are often implicit, multi-issue, and expressed in colloquial or underspecified forms. These characteristics make it difficult for conventional retrieval-augmented generation pipelines to recover the statutory elements required for accurate retrieval. Dense retrievers focus primarily on the literal surface form of the query, whereas lightweight rerankers lack the legal-reasoning capacit

Refer to the [full paper](https://arxiv.org/abs/2601.17692v1) for detailed methodology.