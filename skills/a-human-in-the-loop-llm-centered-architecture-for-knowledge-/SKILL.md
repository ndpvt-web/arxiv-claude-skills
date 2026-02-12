---
name: "a-human-in-the-loop-llm-centered-architecture-for-knowledge-"
description: "Large Language Models (LLMs) excel at language understanding but remain limited in knowledge-intensive domains due to hallucinations, outdated information, and limited explainability. Implements techniques from the paper 'A Human-in-the-Loop, LLM-Centered Architecture for Knowledge-Graph Question Answering' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework), (design & ui) or when the user references techniques from this research area."
---

# A Human-in-the-Loop, LLM-Centered Architecture for Knowledge-Graph Question Answering

**Source:** [https://arxiv.org/abs/2602.05512v2](https://arxiv.org/abs/2602.05512v2)
**Category:** cs.CL | **Published:** 2026-02-05 | **Skill Score:** 77
**Authors:** Larissa Pusch, Alexandre Courtiol, Tim Conrad

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

> Large Language Models (LLMs) excel at language understanding but remain limited in knowledge-intensive domains due to hallucinations, outdated information, and limited explainability. Text-based retrieval-augmented generation (RAG) helps ground model outputs in external sources but struggles with multi-hop reasoning. Knowledge Graphs (KGs), in contrast, support precise, explainable querying, yet require a knowledge of query languages. This work introduces an interactive framework in which LLMs g

Refer to the [full paper](https://arxiv.org/abs/2602.05512v2) for detailed methodology.