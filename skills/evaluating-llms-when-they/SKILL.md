---
name: "evaluating-llms-when-they"
description: "Evaluating mathematical reasoning in LLMs is constrained by limited benchmark sizes and inherent model stochasticity, yielding high-variance accuracy estimates and unstable rankings across platforms. Implements techniques from the paper 'Evaluating LLMs When They Do Not Know the Answer: Statistical Evaluation of Mathematical Reasoning via Comparative Signals' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# Evaluating LLMs When They Do Not Know the Answer: Statistical Evaluation of Mathematical Reasoning via Comparative Signals

**Source:** [https://arxiv.org/abs/2602.03061v1](https://arxiv.org/abs/2602.03061v1)
**Category:** cs.LG | **Published:** 2026-02-03 | **Skill Score:** 69
**Authors:** Zihan Dong, Zhixian Zhang, Yang Zhou...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Leverages:** this observation to design a statistically efficient evaluation framework that combines standard labeled

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

> Evaluating mathematical reasoning in LLMs is constrained by limited benchmark sizes and inherent model stochasticity, yielding high-variance accuracy estimates and unstable rankings across platforms. On difficult problems, an LLM may fail to produce a correct final answer, yet still provide reliable pairwise comparison signals indicating which of two candidate solutions is better. We leverage this observation to design a statistically efficient evaluation framework that combines standard labeled

Refer to the [full paper](https://arxiv.org/abs/2602.03061v1) for detailed methodology.