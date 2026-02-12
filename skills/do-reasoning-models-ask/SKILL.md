---
name: "do-reasoning-models-ask"
description: "Large Language Models (LLMs) excel at many tasks but still struggle with a critical ability for LLM-based agents: asking good questions for resolving ambiguity in user requests. Implements techniques from the paper 'Do Reasoning Models Ask Better Questions? A Formal Information-Theoretic Analysis on Multi-Turn LLM Games' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework) or when the user references techniques from this research area."
---

# Do Reasoning Models Ask Better Questions? A Formal Information-Theoretic Analysis on Multi-Turn LLM Games

**Source:** [https://arxiv.org/abs/2601.17716v1](https://arxiv.org/abs/2601.17716v1)
**Category:** cs.LG | **Published:** 2026-01-25 | **Skill Score:** 67
**Authors:** Daniel M. Pedrozo, Telma W. de L. Soares, Bryan L. M. de Oliveira

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

> Large Language Models (LLMs) excel at many tasks but still struggle with a critical ability for LLM-based agents: asking good questions for resolving ambiguity in user requests. While prior work has explored information-seeking behavior through word games, existing benchmarks lack comprehensive evaluation frameworks that provide both final and intermediate signals based on Information Gain (IG). Moreover, they rarely provide systematic comparisons between models that use chain-of-thought reasoni

Refer to the [full paper](https://arxiv.org/abs/2601.17716v1) for detailed methodology.