---
name: "valueflow-measuring-the-propagation"
description: "Multi-agent large language model (LLM) systems increasingly consist of agents that observe and respond to one another's outputs. Implements techniques from the paper 'ValueFlow: Measuring the Propagation of Value Perturbations in Multi-Agent LLM Systems' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# ValueFlow: Measuring the Propagation of Value Perturbations in Multi-Agent LLM Systems

**Source:** [https://arxiv.org/abs/2602.08567v1](https://arxiv.org/abs/2602.08567v1)
**Category:** cs.MA | **Published:** 2026-02-09 | **Skill Score:** 69
**Authors:** Jinnuo Liu, Chuke Liu, Hua Shen

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

> Multi-agent large language model (LLM) systems increasingly consist of agents that observe and respond to one another's outputs. While value alignment is typically evaluated for isolated models, how value perturbations propagate through agent interactions remains poorly understood. We present ValueFlow, a perturbation-based evaluation framework for measuring and analyzing value drift in multi-agent systems. ValueFlow introduces a 56-value evaluation dataset derived from the Schwartz Value Survey

Refer to the [full paper](https://arxiv.org/abs/2602.08567v1) for detailed methodology.