---
name: "llm-grounded-dynamic-task-planning"
description: "While Large Language Models (LLM) enable non-experts to specify open-world multi-robot tasks, the generated plans often lack kinematic feasibility and are not efficient, especially in long-horizon ... Implements techniques from the paper 'LLM-Grounded Dynamic Task Planning with Hierarchical Temporal Logic for Human-Aware Multi-Robot Collaboration' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# LLM-Grounded Dynamic Task Planning with Hierarchical Temporal Logic for Human-Aware Multi-Robot Collaboration

**Source:** [https://arxiv.org/abs/2602.09472v1](https://arxiv.org/abs/2602.09472v1)
**Category:** cs.RO | **Published:** 2026-02-10 | **Skill Score:** 59
**Authors:** Shuyuan Hu, Tao Lin, Kai Ye...

## Core Capability

Build and orchestrate AI agent workflows.

## Key Techniques

- **Proposed technique:** a neuro-symbolic framework that grounds llm reasoning into hierarchical

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

> While Large Language Models (LLM) enable non-experts to specify open-world multi-robot tasks, the generated plans often lack kinematic feasibility and are not efficient, especially in long-horizon scenarios. Formal methods like Linear Temporal Logic (LTL) offer correctness and optimal guarantees, but are typically confined to static, offline settings and struggle with computational scalability. To bridge this gap, we propose a neuro-symbolic framework that grounds LLM reasoning into hierarchical

Refer to the [full paper](https://arxiv.org/abs/2602.09472v1) for detailed methodology.