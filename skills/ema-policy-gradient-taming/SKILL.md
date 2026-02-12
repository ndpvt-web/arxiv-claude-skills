---
name: "ema-policy-gradient-taming"
description: "Reinforcement Learning (RL) has enabled Large Language Models (LLMs) to acquire increasingly complex reasoning and agentic behaviors. Implements techniques from the paper 'EMA Policy Gradient: Taming Reinforcement Learning for LLMs with EMA Anchor and Top-k KL' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# EMA Policy Gradient: Taming Reinforcement Learning for LLMs with EMA Anchor and Top-k KL

**Source:** [https://arxiv.org/abs/2602.04417v1](https://arxiv.org/abs/2602.04417v1)
**Category:** cs.LG | **Published:** 2026-02-04 | **Skill Score:** 77
**Authors:** Lunjun Zhang, Jimmy Ba

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** two simple techniques to improve policy gradient algorithms for llms
- **Proposed technique:** top-k kl estimator

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

> Reinforcement Learning (RL) has enabled Large Language Models (LLMs) to acquire increasingly complex reasoning and agentic behaviors. In this work, we propose two simple techniques to improve policy gradient algorithms for LLMs. First, we replace the fixed anchor policy during RL with an Exponential Moving Average (EMA), similar to a target network in deep Q-learning. Second, we introduce Top-k KL estimator, which allows for flexible interpolation between exact KL and sampled KL. We derive the s

Refer to the [full paper](https://arxiv.org/abs/2602.04417v1) for detailed methodology.