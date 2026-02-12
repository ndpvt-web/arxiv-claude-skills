---
name: "rc-grpo-rewardconditioned-group-relative"
description: "Multi-turn tool calling is challenging for Large Language Models (LLMs) because rewards are sparse and exploration is expensive. Implements techniques from the paper 'RC-GRPO: Reward-Conditioned Group Relative Policy Optimization for Multi-Turn Tool Calling Agents' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# RC-GRPO: Reward-Conditioned Group Relative Policy Optimization for Multi-Turn Tool Calling Agents

**Source:** [https://arxiv.org/abs/2602.03025v1](https://arxiv.org/abs/2602.03025v1)
**Category:** cs.AI | **Published:** 2026-02-03 | **Skill Score:** 80
**Authors:** Haitian Zhong, Jixiu Zhai, Lei Song...

## Core Capability

Build and orchestrate AI agent workflows.

## Key Techniques

- **Proposed technique:** rc-grpo (reward-conditioned group relative policy optimization)

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

> Multi-turn tool calling is challenging for Large Language Models (LLMs) because rewards are sparse and exploration is expensive. A common recipe, SFT followed by GRPO, can stall when within-group reward variation is low (e.g., more rollouts in a group receive the all 0 or all 1 reward), making the group-normalized advantage uninformative and yielding vanishing updates. To address this problem, we propose RC-GRPO (Reward-Conditioned Group Relative Policy Optimization), which treats exploration as

Refer to the [full paper](https://arxiv.org/abs/2602.03025v1) for detailed methodology.