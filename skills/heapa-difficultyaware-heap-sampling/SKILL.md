---
name: "heapa-difficultyaware-heap-sampling"
description: "RLVR is now a standard way to train LLMs on reasoning tasks with verifiable outcomes, but when rollout generation dominates the cost, efficiency depends heavily on which prompts you sample and when. Implements techniques from the paper 'HeaPA: Difficulty-Aware Heap Sampling and On-Policy Query Augmentation for LLM Reinforcement Learning' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# HeaPA: Difficulty-Aware Heap Sampling and On-Policy Query Augmentation for LLM Reinforcement Learning

**Source:** [https://arxiv.org/abs/2601.22448v1](https://arxiv.org/abs/2601.22448v1)
**Category:** cs.LG | **Published:** 2026-01-30 | **Skill Score:** 66
**Authors:** Weiqi Wang, Xin Liu, Binxuan Huang...

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

> RLVR is now a standard way to train LLMs on reasoning tasks with verifiable outcomes, but when rollout generation dominates the cost, efficiency depends heavily on which prompts you sample and when. In practice, prompt pools are often static or only loosely tied to the model's learning progress, so uniform sampling can't keep up with the shifting capability frontier and ends up wasting rollouts on prompts that are already solved or still out of reach. Existing approaches improve efficiency throu

Refer to the [full paper](https://arxiv.org/abs/2601.22448v1) for detailed methodology.