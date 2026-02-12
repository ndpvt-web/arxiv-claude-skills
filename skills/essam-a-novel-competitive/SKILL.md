---
name: "essam-a-novel-competitive"
description: "Reinforcement learning (RL) has become a key training step for improving mathematical reasoning in large language models (LLMs), but it often has high GPU memory usage, which makes it hard to use i... Implements techniques from the paper 'ESSAM: A Novel Competitive Evolution Strategies Approach to Reinforcement Learning for Memory Efficient LLMs Fine-Tuning' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# ESSAM: A Novel Competitive Evolution Strategies Approach to Reinforcement Learning for Memory Efficient LLMs Fine-Tuning

**Source:** [https://arxiv.org/abs/2602.01003v1](https://arxiv.org/abs/2602.01003v1)
**Category:** cs.LG | **Published:** 2026-02-01 | **Skill Score:** 62
**Authors:** Zhishen Sun, Sizhe Dang, Guang Dai...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** evolution strategies with sharpness-aware maximization (essam)

## Workflow

1. Parse the user's information need or query
2. Formulate effective search strategies
3. Retrieve relevant documents, code, or data
4. Synthesize findings into a coherent response
5. Provide citations and references

## Agent Coordination

- Decompose complex tasks into independent subtasks where possible
- Use parallel execution for independent subtasks
- Implement error recovery and retry logic
- Aggregate partial results when full completion fails

## Research Context

> Reinforcement learning (RL) has become a key training step for improving mathematical reasoning in large language models (LLMs), but it often has high GPU memory usage, which makes it hard to use in settings with limited resources. To reduce these issues, we propose Evolution Strategies with Sharpness-Aware Maximization (ESSAM), a full parameter fine-tuning framework that tightly combines the zero-order search in parameter space from Evolution Strategies (ES) with the Sharpness-Aware Maximizatio

Refer to the [full paper](https://arxiv.org/abs/2602.01003v1) for detailed methodology.