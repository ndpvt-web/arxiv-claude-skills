---
name: "adatir-adaptive-toolintegrated-reasoning"
description: "Tool-Integrated Reasoning (TIR) has significantly enhanced the capabilities of Large Language Models (LLMs), yet current agents tend to exhibit cognitive offloading, redundantly invoking external t... Implements techniques from the paper 'AdaTIR: Adaptive Tool-Integrated Reasoning via Difficulty-Aware Policy Optimization' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework) or when the user references techniques from this research area."
---

# AdaTIR: Adaptive Tool-Integrated Reasoning via Difficulty-Aware Policy Optimization

**Source:** [https://arxiv.org/abs/2601.14696v1](https://arxiv.org/abs/2601.14696v1)
**Category:** cs.CL | **Published:** 2026-01-21 | **Skill Score:** 59
**Authors:** Zhaiyu Fang, Ruipeng Sun

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

> Tool-Integrated Reasoning (TIR) has significantly enhanced the capabilities of Large Language Models (LLMs), yet current agents tend to exhibit cognitive offloading, redundantly invoking external tools even for simple tasks. In this paper, we suggest that true agentic intelligence requires not just tool invocation, but the adaptive wisdom to discern when to use them. We propose AdaTIR, a framework that shifts the paradigm from static tool invocation to difficulty-aware reasoning internalization.

Refer to the [full paper](https://arxiv.org/abs/2601.14696v1) for detailed methodology.