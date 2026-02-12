---
name: "less-noise-more-voice"
description: "Reinforcement Learning with Verifiable Rewards (RLVR) has advanced LLM reasoning, but remains constrained by inefficient exploration under limited rollout budgets, leading to low sampling success a... Implements techniques from the paper 'Less Noise, More Voice: Reinforcement Learning for Reasoning via Instruction Purification' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# Less Noise, More Voice: Reinforcement Learning for Reasoning via Instruction Purification

**Source:** [https://arxiv.org/abs/2601.21244v2](https://arxiv.org/abs/2601.21244v2)
**Category:** cs.LG | **Published:** 2026-01-29 | **Skill Score:** 71
**Authors:** Yiju Guo, Tianyi Hu, Zexu Sun...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** the less noise sampling framework (lens)

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

> Reinforcement Learning with Verifiable Rewards (RLVR) has advanced LLM reasoning, but remains constrained by inefficient exploration under limited rollout budgets, leading to low sampling success and unstable training in complex tasks. We find that many exploration failures arise not from problem difficulty, but from a small number of prompt tokens that introduce interference. Building on this insight, we propose the Less Noise Sampling Framework (LENS), which first prompts by identifying and re

Refer to the [full paper](https://arxiv.org/abs/2601.21244v2) for detailed methodology.