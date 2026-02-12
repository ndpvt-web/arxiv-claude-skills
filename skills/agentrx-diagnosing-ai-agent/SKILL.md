---
name: "agentrx-diagnosing-ai-agent"
description: "AI agents often fail in ways that are difficult to localize because executions are probabilistic, long-horizon, multi-agent, and mediated by noisy tool outputs. Implements techniques from the paper 'AgentRx: Diagnosing AI Agent Failures from Execution Trajectories' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework) or when the user references techniques from this research area."
---

# AgentRx: Diagnosing AI Agent Failures from Execution Trajectories

**Source:** [https://arxiv.org/abs/2602.02475v1](https://arxiv.org/abs/2602.02475v1)
**Category:** cs.AI | **Published:** 2026-02-02 | **Skill Score:** 58
**Authors:** Shraddha Barke, Arnav Goyal, Alind Khare...

## Core Capability

Build and orchestrate AI agent workflows.

## Key Techniques

- **Multi-agent architecture** for task decomposition and parallel execution

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

> AI agents often fail in ways that are difficult to localize because executions are probabilistic, long-horizon, multi-agent, and mediated by noisy tool outputs. We address this gap by manually annotating failed agent runs and release a novel benchmark of 115 failed trajectories spanning structured API workflows, incident management, and open-ended web/file tasks. Each trajectory is annotated with a critical failure step and a category from a grounded-theory derived, cross-domain failure taxonomy

Refer to the [full paper](https://arxiv.org/abs/2602.02475v1) for detailed methodology.