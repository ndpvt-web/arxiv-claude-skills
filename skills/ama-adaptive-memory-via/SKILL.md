---
name: "ama-adaptive-memory-via"
description: "The rapid evolution of Large Language Model (LLM) agents has necessitated robust memory systems to support cohesive long-term interaction and complex reasoning. Implements techniques from the paper 'AMA: Adaptive Memory via Multi-Agent Collaboration' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# AMA: Adaptive Memory via Multi-Agent Collaboration

**Source:** [https://arxiv.org/abs/2601.20352v2](https://arxiv.org/abs/2601.20352v2)
**Category:** cs.AI | **Published:** 2026-01-28 | **Skill Score:** 76
**Authors:** Weiquan Huang, Zixuan Wang, Hehai Lin...

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

> The rapid evolution of Large Language Model (LLM) agents has necessitated robust memory systems to support cohesive long-term interaction and complex reasoning. Benefiting from the strong capabilities of LLMs, recent research focus has shifted from simple context extension to the development of dedicated agentic memory systems. However, existing approaches typically rely on rigid retrieval granularity, accumulation-heavy maintenance strategies, and coarse-grained update mechanisms. These design 

Refer to the [full paper](https://arxiv.org/abs/2601.20352v2) for detailed methodology.