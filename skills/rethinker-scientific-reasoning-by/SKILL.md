---
name: "rethinker-scientific-reasoning-by"
description: "Expert-level scientific reasoning remains challenging for large language models, particularly on benchmarks such as Humanity's Last Exam (HLE), where rigid tool pipelines, brittle multi-agent coord... Implements techniques from the paper 'ReThinker: Scientific Reasoning by Rethinking with Guided Reflection and Confidence Control' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# ReThinker: Scientific Reasoning by Rethinking with Guided Reflection and Confidence Control

**Source:** [https://arxiv.org/abs/2602.04496v1](https://arxiv.org/abs/2602.04496v1)
**Category:** cs.AI | **Published:** 2026-02-04 | **Skill Score:** 83
**Authors:** Zhentao Tang, Yuqi Cui, Shixiong Kai...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Multi-agent architecture** for task decomposition and parallel execution
- **Retrieval-augmented** approach for grounding responses in external knowledge

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

> Expert-level scientific reasoning remains challenging for large language models, particularly on benchmarks such as Humanity's Last Exam (HLE), where rigid tool pipelines, brittle multi-agent coordination, and inefficient test-time scaling often limit performance. We introduce ReThinker, a confidence-aware agentic framework that orchestrates retrieval, tool use, and multi-agent reasoning through a stage-wise Solver-Critic-Selector architecture. Rather than following a fixed pipeline, ReThinker d

Refer to the [full paper](https://arxiv.org/abs/2602.04496v1) for detailed methodology.