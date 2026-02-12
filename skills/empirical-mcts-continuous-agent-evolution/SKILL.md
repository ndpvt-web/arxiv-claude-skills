---
name: "empirical-mcts-continuous-agent-evolution"
description: "Inference-time scaling strategies, particularly Monte Carlo Tree Search (MCTS), have significantly enhanced the reasoning capabilities of Large Language Models (LLMs). Implements techniques from the paper 'Empirical-MCTS: Continuous Agent Evolution via Dual-Experience Monte Carlo Tree Search' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# Empirical-MCTS: Continuous Agent Evolution via Dual-Experience Monte Carlo Tree Search

**Source:** [https://arxiv.org/abs/2602.04248v1](https://arxiv.org/abs/2602.04248v1)
**Category:** cs.AI | **Published:** 2026-02-04 | **Skill Score:** 78
**Authors:** Hao Lu, Haoyuan Huang, Yulin Zhou...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** empirical-mcts

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

> Inference-time scaling strategies, particularly Monte Carlo Tree Search (MCTS), have significantly enhanced the reasoning capabilities of Large Language Models (LLMs). However, current approaches remain predominantly stateless, discarding successful reasoning patterns after each problem instance and failing to mimic the empirical accumulation of wisdom characteristic of human problem-solving. To bridge this gap, we introduce Empirical-MCTS, a dual-loop framework that transforms stateless search 

Refer to the [full paper](https://arxiv.org/abs/2602.04248v1) for detailed methodology.