---
name: "ecco-evidencedriven-causal-reasoning"
description: "Compiler auto-tuning faces a dichotomy between traditional black-box search methods, which lack semantic guidance, and recent Large Language Model (LLM) approaches, which often suffer from superfic... Implements techniques from the paper 'ECCO: Evidence-Driven Causal Reasoning for Compiler Optimization' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# ECCO: Evidence-Driven Causal Reasoning for Compiler Optimization

**Source:** [https://arxiv.org/abs/2602.00087v1](https://arxiv.org/abs/2602.00087v1)
**Category:** cs.LG | **Published:** 2026-01-23 | **Skill Score:** 75
**Authors:** Haolin Pan, Lianghong Huang, Jinyuan Dong...

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

> Compiler auto-tuning faces a dichotomy between traditional black-box search methods, which lack semantic guidance, and recent Large Language Model (LLM) approaches, which often suffer from superficial pattern matching and causal opacity. In this paper, we introduce ECCO, a framework that bridges interpretable reasoning with combinatorial search. We first propose a reverse engineering methodology to construct a Chain-of-Thought dataset, explicitly mapping static code features to verifiable perfor

Refer to the [full paper](https://arxiv.org/abs/2602.00087v1) for detailed methodology.