---
name: "monte-carlo-tree-search"
description: "Automated program repair with large language models remains challenging at the repository level due to long-horizon reasoning requirements and the limitations of autoregressive decoding. Implements techniques from the paper 'Monte Carlo Tree Search for Execution-Guided Program Repair with Large Language Models' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# Monte Carlo Tree Search for Execution-Guided Program Repair with Large Language Models

**Source:** [https://arxiv.org/abs/2602.00129v1](https://arxiv.org/abs/2602.00129v1)
**Category:** cs.LG | **Published:** 2026-01-28 | **Skill Score:** 67
**Authors:** Yixuan Liang

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

> Automated program repair with large language models remains challenging at the repository level due to long-horizon reasoning requirements and the limitations of autoregressive decoding. We present CodePilot, a hybrid framework that integrates Monte Carlo Tree Search (MCTS) with large language models to enable execution-guided program repair for real-world GitHub issues. CodePilot performs hierarchical fault localization from repository to file and function level, explores diverse patch trajecto

Refer to the [full paper](https://arxiv.org/abs/2602.00129v1) for detailed methodology.