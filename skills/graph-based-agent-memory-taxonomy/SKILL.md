---
name: "graph-based-agent-memory-taxonomy"
description: "Memory emerges as the core module in the Large Language Model (LLM)-based agents for long-horizon complex tasks (e.g., multi-turn dialogue, game playing, scientific discovery), where memory can ena... Implements techniques from the paper 'Graph-based Agent Memory: Taxonomy, Techniques, and Applications' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# Graph-based Agent Memory: Taxonomy, Techniques, and Applications

**Source:** [https://arxiv.org/abs/2602.05665v1](https://arxiv.org/abs/2602.05665v1)
**Category:** cs.AI | **Published:** 2026-02-05 | **Skill Score:** 79
**Authors:** Chang Yang, Chuang Zhou, Yilin Xiao...

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

> Memory emerges as the core module in the Large Language Model (LLM)-based agents for long-horizon complex tasks (e.g., multi-turn dialogue, game playing, scientific discovery), where memory can enable knowledge accumulation, iterative reasoning and self-evolution. Among diverse paradigms, graph stands out as a powerful structure for agent memory due to the intrinsic capabilities to model relational dependencies, organize hierarchical information, and support efficient retrieval. This survey pres

Refer to the [full paper](https://arxiv.org/abs/2602.05665v1) for detailed methodology.