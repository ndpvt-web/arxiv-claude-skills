---
name: "cofeh-llmdriven-feature-engineering"
description: "Feature Engineering (FE) is pivotal in automated machine learning (AutoML) but remains a bottleneck for traditional methods, which treat it as a black-box search, operating within rigid, predefined... Implements techniques from the paper 'CoFEH: LLM-driven Feature Engineering Empowered by Collaborative Bayesian Hyperparameter Optimization' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# CoFEH: LLM-driven Feature Engineering Empowered by Collaborative Bayesian Hyperparameter Optimization

**Source:** [https://arxiv.org/abs/2602.09851v1](https://arxiv.org/abs/2602.09851v1)
**Category:** cs.LG | **Published:** 2026-02-10 | **Skill Score:** 60
**Authors:** Beicheng Xu, Keyao Ding, Wei Liu...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Leverages:** semantic reasoning to generate unbounded operators

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

> Feature Engineering (FE) is pivotal in automated machine learning (AutoML) but remains a bottleneck for traditional methods, which treat it as a black-box search, operating within rigid, predefined search spaces and lacking domain awareness. While Large Language Models (LLMs) offer a promising alternative by leveraging semantic reasoning to generate unbounded operators, existing methods fail to construct free-form FE pipelines, remaining confined to isolated subtasks such as feature generation. 

Refer to the [full paper](https://arxiv.org/abs/2602.09851v1) for detailed methodology.