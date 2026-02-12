---
name: "entropy-gated-selective-policy-optimizationtokenlevel"
description: "Hybrid training methods for large language models combine supervised fine tuning (SFT) on expert demonstrations with reinforcement learning (RL) on model rollouts, typically at the sample level. Implements techniques from the paper 'Entropy-Gated Selective Policy Optimization:Token-Level Gradient Allocation for Hybrid Training of Large Language Models' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# Entropy-Gated Selective Policy Optimization:Token-Level Gradient Allocation for Hybrid Training of Large Language Models

**Source:** [https://arxiv.org/abs/2602.03309v1](https://arxiv.org/abs/2602.03309v1)
**Category:** cs.LG | **Published:** 2026-02-03 | **Skill Score:** 58
**Authors:** Yuelin Hu, Zhengxue Cheng, Wei Liu...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** entropy gated selective policy optimization (egspo)

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

> Hybrid training methods for large language models combine supervised fine tuning (SFT) on expert demonstrations with reinforcement learning (RL) on model rollouts, typically at the sample level. We propose Entropy Gated Selective Policy Optimization (EGSPO), a three stage framework that extends sample level mixing with token level gradient modulation.   Stage 1, SFT expert learning, establishes a reliable warm up policy using expert demonstrations with a pure SFT loss. Stage 2, RL rollout genera

Refer to the [full paper](https://arxiv.org/abs/2602.03309v1) for detailed methodology.