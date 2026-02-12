---
name: "evoopt-llm-evolving-industrial-optimization"
description: "Optimization modeling via mixed-integer linear programming (MILP) is fundamental to industrial planning and scheduling, yet translating natural-language requirements into solver-executable models a... Implements techniques from the paper 'EvoOpt-LLM: Evolving industrial optimization models with large language models' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework) or when the user references techniques from this research area."
---

# EvoOpt-LLM: Evolving industrial optimization models with large language models

**Source:** [https://arxiv.org/abs/2602.01082v1](https://arxiv.org/abs/2602.01082v1)
**Category:** cs.AI | **Published:** 2026-02-01 | **Skill Score:** 59
**Authors:** Yiliu He, Tianle Li, Binghao Ji...

## Core Capability

Build and orchestrate AI agent workflows.

## Workflow

1. Decompose complex tasks into subtasks
2. Plan agent coordination and communication patterns
3. Execute subtasks with appropriate tools and models
4. Monitor agent progress and handle failures
5. Aggregate results and present final output

## Agent Coordination

- Decompose complex tasks into independent subtasks where possible
- Use parallel execution for independent subtasks
- Implement error recovery and retry logic
- Aggregate partial results when full completion fails

## Research Context

> Optimization modeling via mixed-integer linear programming (MILP) is fundamental to industrial planning and scheduling, yet translating natural-language requirements into solver-executable models and maintaining them under evolving business rules remains highly expertise-intensive. While large language models (LLMs) offer promising avenues for automation, existing methods often suffer from low data efficiency, limited solver-level validity, and poor scalability to industrial-scale problems. To a

Refer to the [full paper](https://arxiv.org/abs/2602.01082v1) for detailed methodology.