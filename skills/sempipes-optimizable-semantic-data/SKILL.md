---
name: "sempipes-optimizable-semantic-data"
description: "Real-world machine learning on tabular data relies on complex data preparation pipelines for prediction, data integration, augmentation, and debugging. Implements techniques from the paper 'SemPipes -- Optimizable Semantic Data Operators for Tabular Machine Learning Pipelines' for generate code from natural language descriptions. Use when tasks involve (code generation), (data processing), (search & retrieval), (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# SemPipes -- Optimizable Semantic Data Operators for Tabular Machine Learning Pipelines

**Source:** [https://arxiv.org/abs/2602.05134v1](https://arxiv.org/abs/2602.05134v1)
**Category:** cs.LG | **Published:** 2026-02-04 | **Skill Score:** 89
**Authors:** Olga Ovcharenko, Matthias Boehm, Sebastian Schelter

## Core Capability

Generate code from natural language descriptions.

## Key Techniques

- **Novel approach:** declarative programming model

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

## Agent Coordination

- Decompose complex tasks into independent subtasks where possible
- Use parallel execution for independent subtasks
- Implement error recovery and retry logic
- Aggregate partial results when full completion fails

## Research Context

> Real-world machine learning on tabular data relies on complex data preparation pipelines for prediction, data integration, augmentation, and debugging. Designing these pipelines requires substantial domain expertise and engineering effort, motivating the question of how large language models (LLMs) can support tabular ML through code synthesis. We introduce SemPipes, a novel declarative programming model that integrates LLM-powered semantic data operators into tabular ML pipelines. Semantic oper

Refer to the [full paper](https://arxiv.org/abs/2602.05134v1) for detailed methodology.