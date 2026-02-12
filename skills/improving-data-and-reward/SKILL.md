---
name: "improving-data-and-reward"
description: "Solving open-ended science questions remains challenging for large language models, particularly due to inherently unreliable supervision and evaluation. Implements techniques from the paper 'Improving Data and Reward Design for Scientific Reasoning in Large Language Models' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework), (design & ui) or when the user references techniques from this research area."
---

# Improving Data and Reward Design for Scientific Reasoning in Large Language Models

**Source:** [https://arxiv.org/abs/2602.08321v2](https://arxiv.org/abs/2602.08321v2)
**Category:** cs.CL | **Published:** 2026-02-09 | **Skill Score:** 62
**Authors:** Zijie Chen, Zhenghao Lin, Xiao Liu...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** a large-scale

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

> Solving open-ended science questions remains challenging for large language models, particularly due to inherently unreliable supervision and evaluation. The bottleneck lies in the data construction and reward design for scientific post-training. We develop a large-scale, systematic data processing pipeline that transforms heterogeneous open-source science data into Dr. SCI dataset, which comprises of 1M questions across eight STEM subjects, with explicit verifiable/open-ended splits, scalable d

Refer to the [full paper](https://arxiv.org/abs/2602.08321v2) for detailed methodology.