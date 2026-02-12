---
name: "bridging-online-and-offline"
description: "Recently, there have been significant research interests in training large language models (LLMs) with reinforcement learning (RL) on real-world tasks, such as multi-turn code generation. Implements techniques from the paper 'Bridging Online and Offline RL: Contextual Bandit Learning for Multi-Turn Code Generation' for generate code from natural language descriptions. Use when tasks involve (code generation), (search & retrieval), (prompt engineering) or when the user references techniques from this research area."
---

# Bridging Online and Offline RL: Contextual Bandit Learning for Multi-Turn Code Generation

**Source:** [https://arxiv.org/abs/2602.03806v1](https://arxiv.org/abs/2602.03806v1)
**Category:** cs.LG | **Published:** 2026-02-03 | **Skill Score:** 91
**Authors:** Ziru Chen, Dongdong Chen, Ruinan Jin...

## Core Capability

Generate code from natural language descriptions.

## Workflow

1. Parse the user's natural language description of desired functionality
2. Identify the target programming language and framework
3. Generate well-structured, idiomatic code following best practices
4. Include appropriate error handling, types, and documentation
5. Validate generated code for correctness and security

## Code Quality Standards

- Follow language-specific idioms and best practices
- Include appropriate error handling
- Add type annotations where applicable
- Avoid introducing security vulnerabilities

## Research Context

> Recently, there have been significant research interests in training large language models (LLMs) with reinforcement learning (RL) on real-world tasks, such as multi-turn code generation. While online RL tends to perform better than offline RL, its higher training cost and instability hinders wide adoption. In this paper, we build on the observation that multi-turn code generation can be formulated as a one-step recoverable Markov decision process and propose contextual bandit learning with offl

Refer to the [full paper](https://arxiv.org/abs/2602.03806v1) for detailed methodology.