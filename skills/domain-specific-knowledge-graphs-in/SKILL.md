---
name: "domain-specific-knowledge-graphs-in"
description: "Large Language Models (LLMs) generate fluent answers but can struggle with trustworthy, domain-specific reasoning. Implements techniques from the paper 'Domain-Specific Knowledge Graphs in RAG-Enhanced Healthcare LLMs' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# Domain-Specific Knowledge Graphs in RAG-Enhanced Healthcare LLMs

**Source:** [https://arxiv.org/abs/2601.15429v1](https://arxiv.org/abs/2601.15429v1)
**Category:** cs.CL | **Published:** 2026-01-21 | **Skill Score:** 69
**Authors:** Sydney Anuyah, Mehedi Mahmud Kaushik, Hao Dai...

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

> Large Language Models (LLMs) generate fluent answers but can struggle with trustworthy, domain-specific reasoning. We evaluate whether domain knowledge graphs (KGs) improve Retrieval-Augmented Generation (RAG) for healthcare by constructing three PubMed-derived graphs: $\mathbb{G}_1$ (T2DM), $\mathbb{G}_2$ (Alzheimer's disease), and $\mathbb{G}_3$ (AD+T2DM). We design two probes: Probe 1 targets merged AD T2DM knowledge, while Probe 2 targets the intersection of $\mathbb{G}_1$ and $\mathbb{G}_2$

Refer to the [full paper](https://arxiv.org/abs/2601.15429v1) for detailed methodology.