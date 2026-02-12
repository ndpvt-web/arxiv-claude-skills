---
name: "d-core-incentivizing-task-decomposition"
description: "Effective tool use and reasoning are essential capabilities for large reasoning models~(LRMs) to address complex real-world problems. Implements techniques from the paper 'D-CORE: Incentivizing Task Decomposition in Large Reasoning Models for Complex Tool Use' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework) or when the user references techniques from this research area."
---

# D-CORE: Incentivizing Task Decomposition in Large Reasoning Models for Complex Tool Use

**Source:** [https://arxiv.org/abs/2602.02160v1](https://arxiv.org/abs/2602.02160v1)
**Category:** cs.CL | **Published:** 2026-02-02 | **Skill Score:** 66
**Authors:** Bowen Xu, Shaoyu Wu, Hao Jiang...

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

> Effective tool use and reasoning are essential capabilities for large reasoning models~(LRMs) to address complex real-world problems. Through empirical analysis, we identify that current LRMs lack the capability of sub-task decomposition in complex tool use scenarios, leading to Lazy Reasoning. To address this, we propose a two-stage training framework D-CORE~(\underline{\textbf{D}}ecomposing tasks and \underline{\textbf{Co}}mposing \underline{\textbf{Re}}asoning processes) that first incentiviz

Refer to the [full paper](https://arxiv.org/abs/2602.02160v1) for detailed methodology.