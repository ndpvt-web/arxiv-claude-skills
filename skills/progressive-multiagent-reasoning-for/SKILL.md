---
name: "progressive-multiagent-reasoning-for"
description: "Predicting gene regulation responses to biological perturbations requires reasoning about underlying biological causalities. Implements techniques from the paper 'Progressive Multi-Agent Reasoning for Biological Perturbation Prediction' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework) or when the user references techniques from this research area."
---

# Progressive Multi-Agent Reasoning for Biological Perturbation Prediction

**Source:** [https://arxiv.org/abs/2602.07408v1](https://arxiv.org/abs/2602.07408v1)
**Category:** cs.AI | **Published:** 2026-02-07 | **Skill Score:** 59
**Authors:** Hyomin Kim, Sang-Yeon Hwang, Jaechang Lim...

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

> Predicting gene regulation responses to biological perturbations requires reasoning about underlying biological causalities. While large language models (LLMs) show promise for such tasks, they are often overwhelmed by the entangled nature of high-dimensional perturbation results. Moreover, recent works have primarily focused on genetic perturbations in single-cell experiments, leaving bulk-cell chemical perturbations, which is central to drug discovery, largely unexplored. Motivated by this, we

Refer to the [full paper](https://arxiv.org/abs/2602.07408v1) for detailed methodology.