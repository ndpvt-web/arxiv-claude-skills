---
name: "ttcs-testtime-curriculum-synthesis"
description: "Test-Time Training offers a promising way to improve the reasoning ability of large language models (LLMs) by adapting the model using only the test questions. Implements techniques from the paper 'TTCS: Test-Time Curriculum Synthesis for Self-Evolving' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework) or when the user references techniques from this research area."
---

# TTCS: Test-Time Curriculum Synthesis for Self-Evolving

**Source:** [https://arxiv.org/abs/2601.22628v1](https://arxiv.org/abs/2601.22628v1)
**Category:** cs.LG | **Published:** 2026-01-30 | **Skill Score:** 85
**Authors:** Chengyi Yang, Zhishang Xiang, Yunbo Tang...

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

> Test-Time Training offers a promising way to improve the reasoning ability of large language models (LLMs) by adapting the model using only the test questions. However, existing methods struggle with difficult reasoning problems for two reasons: raw test questions are often too difficult to yield high-quality pseudo-labels, and the limited size of test sets makes continuous online updates prone to instability. To address these limitations, we propose TTCS, a co-evolving test-time training framew

Refer to the [full paper](https://arxiv.org/abs/2601.22628v1) for detailed methodology.