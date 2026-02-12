---
name: "magellan-autonomous-discovery-of"
description: "Modern compilers rely on hand-crafted heuristics to guide optimization passes. Implements techniques from the paper 'Magellan: Autonomous Discovery of Novel Compiler Optimization Heuristics with AlphaEvolve' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# Magellan: Autonomous Discovery of Novel Compiler Optimization Heuristics with AlphaEvolve

**Source:** [https://arxiv.org/abs/2601.21096v1](https://arxiv.org/abs/2601.21096v1)
**Category:** cs.AI | **Published:** 2026-01-28 | **Skill Score:** 73
**Authors:** Hongzheng Chen, Alexander Novikov, Ngân Vũ...

## Core Capability

Search, retrieve, and synthesize information.

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

> Modern compilers rely on hand-crafted heuristics to guide optimization passes. These human-designed rules often struggle to adapt to the complexity of modern software and hardware and lead to high maintenance burden. To address this challenge, we present Magellan, an agentic framework that evolves the compiler pass itself by synthesizing executable C++ decision logic. Magellan couples an LLM coding agent with evolutionary search and autotuning in a closed loop of generation, evaluation on user-p

Refer to the [full paper](https://arxiv.org/abs/2601.21096v1) for detailed methodology.