---
name: "learning-to-evict-from"
description: "The growing size of Large Language Models (LLMs) makes efficient inference challenging, primarily due to the memory demands of the autoregressive Key-Value (KV) cache. Implements techniques from the paper 'Learning to Evict from Key-Value Cache' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# Learning to Evict from Key-Value Cache

**Source:** [https://arxiv.org/abs/2602.10238v1](https://arxiv.org/abs/2602.10238v1)
**Category:** cs.CL | **Published:** 2026-02-10 | **Skill Score:** 65
**Authors:** Luca Moschella, Laura Manduchi, Ozan Sener

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

> The growing size of Large Language Models (LLMs) makes efficient inference challenging, primarily due to the memory demands of the autoregressive Key-Value (KV) cache. Existing eviction or compression methods reduce cost but rely on heuristics, such as recency or past attention scores, which serve only as indirect proxies for a token's future utility and introduce computational overhead. We reframe KV cache eviction as a reinforcement learning (RL) problem: learning to rank tokens by their predi

Refer to the [full paper](https://arxiv.org/abs/2602.10238v1) for detailed methodology.