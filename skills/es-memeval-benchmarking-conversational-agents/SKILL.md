---
name: "es-memeval-benchmarking-conversational-agents"
description: "Large Language Models (LLMs) have shown strong potential as conversational agents. Implements techniques from the paper 'ES-MemEval: Benchmarking Conversational Agents on Personalized Long-Term Emotional Support' for generate and maintain documentation. Use when tasks involve (documentation), (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# ES-MemEval: Benchmarking Conversational Agents on Personalized Long-Term Emotional Support

**Source:** [https://arxiv.org/abs/2602.01885v1](https://arxiv.org/abs/2602.01885v1)
**Category:** cs.CL | **Published:** 2026-02-02 | **Skill Score:** 86
**Authors:** Tiantian Chen, Jiaqi Lu, Ying Shen...

## Core Capability

Generate and maintain documentation.

## Key Techniques

- **Retrieval-augmented** approach for grounding responses in external knowledge

## Workflow

1. Analyze the codebase to understand architecture and APIs
2. Generate clear, accurate documentation in the appropriate format
3. Include code examples and usage patterns
4. Maintain consistency with existing documentation style

## Agent Coordination

- Decompose complex tasks into independent subtasks where possible
- Use parallel execution for independent subtasks
- Implement error recovery and retry logic
- Aggregate partial results when full completion fails

## Research Context

> Large Language Models (LLMs) have shown strong potential as conversational agents. Yet, their effectiveness remains limited by deficiencies in robust long-term memory, particularly in complex, long-term web-based services such as online emotional support. However, existing long-term dialogue benchmarks primarily focus on static and explicit fact retrieval, failing to evaluate agents in critical scenarios where user information is dispersed, implicit, and continuously evolving. To address this ga

Refer to the [full paper](https://arxiv.org/abs/2602.01885v1) for detailed methodology.