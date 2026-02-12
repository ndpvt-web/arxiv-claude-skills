---
name: "rpo-rag-aligning-small-llms"
description: "Large Language Models (LLMs) have recently demonstrated remarkable reasoning abilities, yet hallucinate on knowledge-intensive tasks. Implements techniques from the paper 'RPO-RAG: Aligning Small LLMs with Relation-aware Preference Optimization for Knowledge Graph Question Answering' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# RPO-RAG: Aligning Small LLMs with Relation-aware Preference Optimization for Knowledge Graph Question Answering

**Source:** [https://arxiv.org/abs/2601.19225v2](https://arxiv.org/abs/2601.19225v2)
**Category:** cs.CL | **Published:** 2026-01-27 | **Skill Score:** 84
**Authors:** Kaehyun Um, KyuHwan Yeom, Haerim Yang...

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

> Large Language Models (LLMs) have recently demonstrated remarkable reasoning abilities, yet hallucinate on knowledge-intensive tasks. Retrieval-augmented generation (RAG) mitigates this issue by grounding answers in external sources, e.g., knowledge graphs (KGs). However, existing KG-based RAG approaches rely on semantics-unaware path sampling and are weakly aligned with KG reasoning objectives, which limits further accuracy gains. They also feed retrieved paths directly into the reasoner withou

Refer to the [full paper](https://arxiv.org/abs/2601.19225v2) for detailed methodology.