---
name: "logicskills-a-structured-benchmark"
description: "Large language models have demonstrated notable performance across various logical reasoning benchmarks. Implements techniques from the paper 'LogicSkills: A Structured Benchmark for Formal Reasoning in Large Language Models' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# LogicSkills: A Structured Benchmark for Formal Reasoning in Large Language Models

**Source:** [https://arxiv.org/abs/2602.06533v1](https://arxiv.org/abs/2602.06533v1)
**Category:** cs.AI | **Published:** 2026-02-06 | **Skill Score:** 58
**Authors:** Brian Rabern, Philipp Mondorf, Barbara Plank

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** logicskills

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

> Large language models have demonstrated notable performance across various logical reasoning benchmarks. However, it remains unclear which core logical skills they truly master. To address this, we introduce LogicSkills, a unified benchmark designed to isolate three fundamental skills in formal reasoning: (i) $\textit{formal symbolization}\unicode{x2014}$translating premises into first-order logic; (ii) $\textit{countermodel construction}\unicode{x2014}$formulating a finite structure in which al

Refer to the [full paper](https://arxiv.org/abs/2602.06533v1) for detailed methodology.