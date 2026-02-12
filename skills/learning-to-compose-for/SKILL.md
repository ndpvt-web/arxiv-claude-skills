---
name: "learning-to-compose-for"
description: "Automatically generating agentic workflows -- executable operator graphs or codes that orchestrate reasoning, verification, and repair -- has become a practical way to solve complex tasks beyond wh... Implements techniques from the paper 'Learning to Compose for Cross-domain Agentic Workflow Generation' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework) or when the user references techniques from this research area."
---

# Learning to Compose for Cross-domain Agentic Workflow Generation

**Source:** [https://arxiv.org/abs/2602.11114v1](https://arxiv.org/abs/2602.11114v1)
**Category:** cs.MA | **Published:** 2026-02-11 | **Skill Score:** 76
**Authors:** Jialiang Wang, Shengxiang Xu, Hanmo Liu...

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

> Automatically generating agentic workflows -- executable operator graphs or codes that orchestrate reasoning, verification, and repair -- has become a practical way to solve complex tasks beyond what single-pass LLM generation can reliably handle. Yet what constitutes a good workflow depends heavily on the task distribution and the available operators. Under domain shift, current systems typically rely on iterative workflow refinement to discover a feasible workflow from a large workflow space, 

Refer to the [full paper](https://arxiv.org/abs/2602.11114v1) for detailed methodology.