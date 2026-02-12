---
name: "deltaevolve-accelerating-scientific-discovery"
description: "LLM-driven evolutionary systems have shown promise for automated science discovery, yet existing approaches such as AlphaEvolve rely on full-code histories that are context-inefficient and potentia... Implements techniques from the paper 'DeltaEvolve: Accelerating Scientific Discovery through Momentum-Driven Evolution' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework), (database & query), (design & ui) or when the user references techniques from this research area."
---

# DeltaEvolve: Accelerating Scientific Discovery through Momentum-Driven Evolution

**Source:** [https://arxiv.org/abs/2602.02919v1](https://arxiv.org/abs/2602.02919v1)
**Category:** cs.AI | **Published:** 2026-02-02 | **Skill Score:** 67
**Authors:** Jiachen Jiang, Tianyu Ding, Zhihui Zhu

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

> LLM-driven evolutionary systems have shown promise for automated science discovery, yet existing approaches such as AlphaEvolve rely on full-code histories that are context-inefficient and potentially provide weak evolutionary guidance. In this work, we first formalize the evolutionary agents as a general Expectation-Maximization framework, where the language model samples candidate programs (E-step) and the system updates the control context based on evaluation feedback (M-step). Under this vie

Refer to the [full paper](https://arxiv.org/abs/2602.02919v1) for detailed methodology.