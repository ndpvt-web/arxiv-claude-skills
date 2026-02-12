---
name: "good-sft-optimizes-for"
description: "Post-training of reasoning LLMs is a holistic process that typically consists of an offline SFT stage followed by an online reinforcement learning (RL) stage. Implements techniques from the paper 'Good SFT Optimizes for SFT, Better SFT Prepares for Reinforcement Learning' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework) or when the user references techniques from this research area."
---

# Good SFT Optimizes for SFT, Better SFT Prepares for Reinforcement Learning

**Source:** [https://arxiv.org/abs/2602.01058v1](https://arxiv.org/abs/2602.01058v1)
**Category:** cs.LG | **Published:** 2026-02-01 | **Skill Score:** 61
**Authors:** Dylan Zhang, Yufeng Xu, Haojin Wang...

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

> Post-training of reasoning LLMs is a holistic process that typically consists of an offline SFT stage followed by an online reinforcement learning (RL) stage. However, SFT is often optimized in isolation to maximize SFT performance alone.   We show that, after identical RL training, models initialized from stronger SFT checkpoints can significantly underperform those initialized from weaker ones. We attribute this to a mismatch typical in current SFT-RL pipelines: the distribution that generates

Refer to the [full paper](https://arxiv.org/abs/2602.01058v1) for detailed methodology.