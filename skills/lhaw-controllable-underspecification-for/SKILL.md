---
name: "lhaw-controllable-underspecification-for"
description: "Long-horizon workflow agents that operate effectively over extended periods are essential for truly autonomous systems. Implements techniques from the paper 'LHAW: Controllable Underspecification for Long-Horizon Tasks' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework) or when the user references techniques from this research area."
---

# LHAW: Controllable Underspecification for Long-Horizon Tasks

**Source:** [https://arxiv.org/abs/2602.10525v1](https://arxiv.org/abs/2602.10525v1)
**Category:** cs.CL | **Published:** 2026-02-11 | **Skill Score:** 66
**Authors:** George Pu, Michael S. Lee, Udari Madhushani Sehwag...

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

> Long-horizon workflow agents that operate effectively over extended periods are essential for truly autonomous systems. Their reliable execution critically depends on the ability to reason through ambiguous situations in which clarification seeking is necessary to ensure correct task execution. However, progress is limited by the lack of scalable, task-agnostic frameworks for systematically curating and measuring the impact of ambiguity across custom workflows. We address this gap by introducing

Refer to the [full paper](https://arxiv.org/abs/2602.10525v1) for detailed methodology.