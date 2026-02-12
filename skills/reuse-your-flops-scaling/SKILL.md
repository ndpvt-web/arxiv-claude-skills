---
name: "reuse-your-flops-scaling"
description: "Typical reinforcement learning (RL) methods for LLM reasoning waste compute on hard problems, where correct on-policy traces are rare, policy gradients vanish, and learning stalls. Implements techniques from the paper 'Reuse your FLOPs: Scaling RL on Hard Problems by Conditioning on Very Off-Policy Prefixes' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework) or when the user references techniques from this research area."
---

# Reuse your FLOPs: Scaling RL on Hard Problems by Conditioning on Very Off-Policy Prefixes

**Source:** [https://arxiv.org/abs/2601.18795v2](https://arxiv.org/abs/2601.18795v2)
**Category:** cs.LG | **Published:** 2026-01-26 | **Skill Score:** 62
**Authors:** Amrith Setlur, Zijian Wang, Andrew Cohen...

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

> Typical reinforcement learning (RL) methods for LLM reasoning waste compute on hard problems, where correct on-policy traces are rare, policy gradients vanish, and learning stalls. To bootstrap more efficient RL, we consider reusing old sampling FLOPs (from prior inference or RL training) in the form of off-policy traces. Standard off-policy methods supervise against off-policy data, causing instabilities during RL optimization. We introduce PrefixRL, where we condition on the prefix of successf

Refer to the [full paper](https://arxiv.org/abs/2601.18795v2) for detailed methodology.