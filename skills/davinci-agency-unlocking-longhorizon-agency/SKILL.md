---
name: "davinci-agency-unlocking-longhorizon-agency"
description: "While Large Language Models (LLMs) excel at short-term tasks, scaling them to long-horizon agentic workflows remains challenging. Implements techniques from the paper 'daVinci-Agency: Unlocking Long-Horizon Agency Data-Efficiently' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# daVinci-Agency: Unlocking Long-Horizon Agency Data-Efficiently

**Source:** [https://arxiv.org/abs/2602.02619v2](https://arxiv.org/abs/2602.02619v2)
**Category:** cs.LG | **Published:** 2026-02-02 | **Skill Score:** 93
**Authors:** Mohan Jiang, Dayuan Fu, Junhao Shi...

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

> While Large Language Models (LLMs) excel at short-term tasks, scaling them to long-horizon agentic workflows remains challenging. The core bottleneck lies in the scarcity of training data that captures authentic long-dependency structures and cross-stage evolutionary dynamics--existing synthesis methods either confine to single-feature scenarios constrained by model distribution, or incur prohibitive human annotation costs, failing to provide scalable, high-quality supervision. We address this b

Refer to the [full paper](https://arxiv.org/abs/2602.02619v2) for detailed methodology.