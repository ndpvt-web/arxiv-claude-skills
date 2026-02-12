---
name: "kglamp-knowledge-graphguided-language"
description: "Heterogeneous multi-robot systems are increasingly deployed in long-horizon missions that require coordination among robots with diverse capabilities. Implements techniques from the paper 'KGLAMP: Knowledge Graph-guided Language model for Adaptive Multi-robot Planning and Replanning' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework) or when the user references techniques from this research area."
---

# KGLAMP: Knowledge Graph-guided Language model for Adaptive Multi-robot Planning and Replanning

**Source:** [https://arxiv.org/abs/2602.04129v1](https://arxiv.org/abs/2602.04129v1)
**Category:** cs.RO | **Published:** 2026-02-04 | **Skill Score:** 64
**Authors:** Chak Lam Shek, Faizan M. Tariq, Sangjae Bae...

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

> Heterogeneous multi-robot systems are increasingly deployed in long-horizon missions that require coordination among robots with diverse capabilities. However, existing planning approaches struggle to construct accurate symbolic representations and maintain plan consistency in dynamic environments. Classical PDDL planners require manually crafted symbolic models, while LLM-based planners often ignore agent heterogeneity and environmental uncertainty. We introduce KGLAMP, a knowledge-graph-guided

Refer to the [full paper](https://arxiv.org/abs/2602.04129v1) for detailed methodology.