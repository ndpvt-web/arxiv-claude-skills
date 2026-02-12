---
name: "when-domains-interact-asymmetric"
description: "Group Relative Policy Optimization (GRPO) has become a key technique for improving reasoning abilities in large language models, yet its behavior under different domain sequencing strategies is poo... Implements techniques from the paper 'When Domains Interact: Asymmetric and Order-Sensitive Cross-Domain Effects in Reinforcement Learning for Reasoning' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework) or when the user references techniques from this research area."
---

# When Domains Interact: Asymmetric and Order-Sensitive Cross-Domain Effects in Reinforcement Learning for Reasoning

**Source:** [https://arxiv.org/abs/2602.01365v1](https://arxiv.org/abs/2602.01365v1)
**Category:** cs.LG | **Published:** 2026-02-01 | **Skill Score:** 58
**Authors:** Wang Yang, Shouren Wang, Chaoda Song...

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

> Group Relative Policy Optimization (GRPO) has become a key technique for improving reasoning abilities in large language models, yet its behavior under different domain sequencing strategies is poorly understood. In particular, the impact of sequential (one domain at a time) versus mixed-domain (multiple domain at a time) training in GRPO has not been systematically studied. We provide the first systematic analysis of training-order effects across math, science, logic, and puzzle reasoning tasks

Refer to the [full paper](https://arxiv.org/abs/2602.01365v1) for detailed methodology.