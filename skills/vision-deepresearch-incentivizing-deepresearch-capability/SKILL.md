---
name: "vision-deepresearch-incentivizing-deepresearch-capability"
description: "Multimodal large language models (MLLMs) have achieved remarkable success across a broad range of vision tasks. Implements techniques from the paper 'Vision-DeepResearch: Incentivizing DeepResearch Capability in Multimodal Large Language Models' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# Vision-DeepResearch: Incentivizing DeepResearch Capability in Multimodal Large Language Models

**Source:** [https://arxiv.org/abs/2601.22060v1](https://arxiv.org/abs/2601.22060v1)
**Category:** cs.CV | **Published:** 2026-01-29 | **Skill Score:** 76
**Authors:** Wenxuan Huang, Yu Zeng, Qiuchen Wang...

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

> Multimodal large language models (MLLMs) have achieved remarkable success across a broad range of vision tasks. However, constrained by the capacity of their internal world knowledge, prior work has proposed augmenting MLLMs by ``reasoning-then-tool-call'' for visual and textual search engines to obtain substantial gains on tasks requiring extensive factual information. However, these approaches typically define multimodal search in a naive setting, assuming that a single full-level or entity-le

Refer to the [full paper](https://arxiv.org/abs/2601.22060v1) for detailed methodology.