---
name: "jet-rl-enabling-onpolicy-fp8"
description: "Reinforcement learning (RL) is essential for enhancing the complex reasoning capabilities of large language models (LLMs). Implements techniques from the paper 'Jet-RL: Enabling On-Policy FP8 Reinforcement Learning with Unified Training and Rollout Precision Flow' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework) or when the user references techniques from this research area."
---

# Jet-RL: Enabling On-Policy FP8 Reinforcement Learning with Unified Training and Rollout Precision Flow

**Source:** [https://arxiv.org/abs/2601.14243v2](https://arxiv.org/abs/2601.14243v2)
**Category:** cs.LG | **Published:** 2026-01-20 | **Skill Score:** 62
**Authors:** Haocheng Xi, Charlie Ruan, Peiyuan Liao...

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

> Reinforcement learning (RL) is essential for enhancing the complex reasoning capabilities of large language models (LLMs). However, existing RL training pipelines are computationally inefficient and resource-intensive, with the rollout phase accounting for over 70% of total training time. Quantized RL training, particularly using FP8 precision, offers a promising approach to mitigating this bottleneck. A commonly adopted strategy applies FP8 precision during rollout while retaining BF16 precisio

Refer to the [full paper](https://arxiv.org/abs/2601.14243v2) for detailed methodology.