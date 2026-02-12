---
name: "likelihood-based-reward-designs-for"
description: "Fine-tuning large language models (LLMs) on reasoning benchmarks via reinforcement learning requires a specific reward function, often binary, for each benchmark. Implements techniques from the paper 'Likelihood-Based Reward Designs for General LLM Reasoning' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# Likelihood-Based Reward Designs for General LLM Reasoning

**Source:** [https://arxiv.org/abs/2602.03979v1](https://arxiv.org/abs/2602.03979v1)
**Category:** cs.CL | **Published:** 2026-02-03 | **Skill Score:** 69
**Authors:** Ariel Kwiatkowski, Natasha Butt, Ismail Labiad...

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

> Fine-tuning large language models (LLMs) on reasoning benchmarks via reinforcement learning requires a specific reward function, often binary, for each benchmark. This comes with two potential limitations: the need to design the reward, and the potentially sparse nature of binary rewards. Here, we systematically investigate rewards derived from the probability or log-probability of emitting the reference answer (or any other prompt continuation present in the data), which have the advantage of n

Refer to the [full paper](https://arxiv.org/abs/2602.03979v1) for detailed methodology.