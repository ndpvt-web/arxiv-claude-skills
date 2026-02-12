---
name: "compactrag-reducing-llm-calls"
description: "Retrieval-augmented generation (RAG) has become a key paradigm for knowledge-intensive question answering. Implements techniques from the paper 'CompactRAG: Reducing LLM Calls and Token Overhead in Multi-Hop Question Answering' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# CompactRAG: Reducing LLM Calls and Token Overhead in Multi-Hop Question Answering

**Source:** [https://arxiv.org/abs/2602.05728v1](https://arxiv.org/abs/2602.05728v1)
**Category:** cs.CL | **Published:** 2026-02-05 | **Skill Score:** 78
**Authors:** Hao Yang, Zhiyu Yang, Xupeng Zhang...

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

> Retrieval-augmented generation (RAG) has become a key paradigm for knowledge-intensive question answering. However, existing multi-hop RAG systems remain inefficient, as they alternate between retrieval and reasoning at each step, resulting in repeated LLM calls, high token consumption, and unstable entity grounding across hops. We propose CompactRAG, a simple yet effective framework that decouples offline corpus restructuring from online reasoning.   In the offline stage, an LLM reads the corpu

Refer to the [full paper](https://arxiv.org/abs/2602.05728v1) for detailed methodology.