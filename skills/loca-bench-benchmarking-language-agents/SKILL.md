---
name: "loca-bench-benchmarking-language-agents"
description: "Large language models (LLMs) are increasingly capable of carrying out long-running, real-world tasks. Implements techniques from the paper 'LOCA-bench: Benchmarking Language Agents Under Controllable and Extreme Context Growth' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# LOCA-bench: Benchmarking Language Agents Under Controllable and Extreme Context Growth

**Source:** [https://arxiv.org/abs/2602.07962v1](https://arxiv.org/abs/2602.07962v1)
**Category:** cs.AI | **Published:** 2026-02-08 | **Skill Score:** 76
**Authors:** Weihao Zeng, Yuzhen Huang, Junxian He

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

> Large language models (LLMs) are increasingly capable of carrying out long-running, real-world tasks. However, as the amount of context grows, their reliability often deteriorates, a phenomenon known as "context rot". Existing long-context benchmarks primarily focus on single-step settings that evaluate a model's ability to retrieve information from a long snippet. In realistic scenarios, however, LLMs often need to act as agents that explore environments, follow instructions and plans, extract 

Refer to the [full paper](https://arxiv.org/abs/2602.07962v1) for detailed methodology.