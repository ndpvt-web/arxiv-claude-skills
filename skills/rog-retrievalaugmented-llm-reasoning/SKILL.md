---
name: "rog-retrievalaugmented-llm-reasoning"
description: "Answering first-order logic (FOL) queries over incomplete knowledge graphs (KGs) is difficult, especially for complex query structures that compose projection, intersection, union, and negation. Implements techniques from the paper 'ROG: Retrieval-Augmented LLM Reasoning for Complex First-Order Queries over Knowledge Graphs' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# ROG: Retrieval-Augmented LLM Reasoning for Complex First-Order Queries over Knowledge Graphs

**Source:** [https://arxiv.org/abs/2602.02382v1](https://arxiv.org/abs/2602.02382v1)
**Category:** cs.CL | **Published:** 2026-02-02 | **Skill Score:** 59
**Authors:** Ziyan Zhang, Chao Wang, Zhuo Chen...

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

> Answering first-order logic (FOL) queries over incomplete knowledge graphs (KGs) is difficult, especially for complex query structures that compose projection, intersection, union, and negation. We propose ROG, a retrieval-augmented framework that combines query-aware neighborhood retrieval with large language model (LLM) chain-of-thought reasoning. ROG decomposes a multi-operator query into a sequence of single-operator sub-queries and grounds each step in compact, query-relevant neighborhood e

Refer to the [full paper](https://arxiv.org/abs/2602.02382v1) for detailed methodology.