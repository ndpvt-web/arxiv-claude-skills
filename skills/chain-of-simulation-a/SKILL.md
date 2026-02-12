---
name: "chain-of-simulation-a"
description: "We present Chain of Simulation (CoS), a novel dual-mode reasoning framework that dynamically routes problems to specialized reasoning strategies in Large Language Models (LLMs). Implements techniques from the paper 'Chain of Simulation: A Dual-Mode Reasoning Framework for Large Language Models with Dynamic Problem Routing' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# Chain of Simulation: A Dual-Mode Reasoning Framework for Large Language Models with Dynamic Problem Routing

**Source:** [https://arxiv.org/abs/2602.02842v1](https://arxiv.org/abs/2602.02842v1)
**Category:** cs.AI | **Published:** 2026-02-02 | **Skill Score:** 74
**Authors:** Saeid Sheikhi

## Core Capability

Build and orchestrate AI agent workflows.

## Key Techniques

- **Proposed technique:** chain of simulation (cos)
- **Novel approach:** dual-mode reasoning framework that dynamically routes problems to specialized reasoning strategies in large language model

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

> We present Chain of Simulation (CoS), a novel dual-mode reasoning framework that dynamically routes problems to specialized reasoning strategies in Large Language Models (LLMs). Unlike existing uniform prompting approaches, CoS employs three distinct reasoning modes: (1) computational flow with self-consistency for mathematical problems, (2) symbolic state tracking with JSON representations for spatial reasoning, and (3) hybrid fact-extraction for multi-hop inference. Through comprehensive evalu

Refer to the [full paper](https://arxiv.org/abs/2602.02842v1) for detailed methodology.