---
name: "learning-to-share-selective"
description: "Agentic systems solve complex tasks by coordinating multiple agents that iteratively reason, invoke tools, and exchange intermediate results. Implements techniques from the paper 'Learning to Share: Selective Memory for Efficient Parallel Agentic Systems' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework) or when the user references techniques from this research area."
---

# Learning to Share: Selective Memory for Efficient Parallel Agentic Systems

**Source:** [https://arxiv.org/abs/2602.05965v1](https://arxiv.org/abs/2602.05965v1)
**Category:** cs.MA | **Published:** 2026-02-05 | **Skill Score:** 64
**Authors:** Joseph Fioresi, Parth Parag Kulkarni, Ashmal Vayani...

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

> Agentic systems solve complex tasks by coordinating multiple agents that iteratively reason, invoke tools, and exchange intermediate results. To improve robustness and solution quality, recent approaches deploy multiple agent teams running in parallel to explore diverse reasoning trajectories. However, parallel execution comes at a significant computational cost: when different teams independently reason about similar sub-problems or execute analogous steps, they repeatedly perform substantial o

Refer to the [full paper](https://arxiv.org/abs/2602.05965v1) for detailed methodology.