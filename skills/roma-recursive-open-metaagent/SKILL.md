---
name: "roma-recursive-open-metaagent"
description: "Current agentic frameworks underperform on long-horizon tasks. Implements techniques from the paper 'ROMA: Recursive Open Meta-Agent Framework for Long-Horizon Multi-Agent Systems' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework), (prompt engineering), (design & ui) or when the user references techniques from this research area."
---

# ROMA: Recursive Open Meta-Agent Framework for Long-Horizon Multi-Agent Systems

**Source:** [https://arxiv.org/abs/2602.01848v1](https://arxiv.org/abs/2602.01848v1)
**Category:** cs.AI | **Published:** 2026-02-02 | **Skill Score:** 84
**Authors:** Salaheddin Alzu'bi, Baran Nama, Arda Kaz...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** roma (recursive open meta-agents)

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

> Current agentic frameworks underperform on long-horizon tasks. As reasoning depth increases, sequential orchestration becomes brittle, context windows impose hard limits that degrade performance, and opaque execution traces make failures difficult to localize or debug. We introduce ROMA (Recursive Open Meta-Agents), a domain-agnostic framework that addresses these limitations through recursive task decomposition and structured aggregation. ROMA decomposes goals into dependency-aware subtask tree

Refer to the [full paper](https://arxiv.org/abs/2602.01848v1) for detailed methodology.