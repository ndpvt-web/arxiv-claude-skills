---
name: "reasoning-and-tooluse-compete"
description: "Agentic Reinforcement Learning (ARL) focuses on training large language models (LLMs) to interleave reasoning with external tool execution to solve complex tasks. Implements techniques from the paper 'Reasoning and Tool-use Compete in Agentic RL:From Quantifying Interference to Disentangled Tuning' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# Reasoning and Tool-use Compete in Agentic RL:From Quantifying Interference to Disentangled Tuning

**Source:** [https://arxiv.org/abs/2602.00994v1](https://arxiv.org/abs/2602.00994v1)
**Category:** cs.AI | **Published:** 2026-02-01 | **Skill Score:** 80
**Authors:** Yu Li, Mingyang Yi, Xiuyu Li...

## Core Capability

Search, retrieve, and synthesize information.

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

> Agentic Reinforcement Learning (ARL) focuses on training large language models (LLMs) to interleave reasoning with external tool execution to solve complex tasks. Most existing ARL methods train a single shared model parameters to support both reasoning and tool use behaviors, implicitly assuming that joint training leads to improved overall agent performance. Despite its widespread adoption, this assumption has rarely been examined empirically. In this paper, we systematically investigate this 

Refer to the [full paper](https://arxiv.org/abs/2602.00994v1) for detailed methodology.