---
name: "bimanibench-a-hierarchical-benchmark"
description: "Multimodal Large Language Models (MLLMs) have significantly advanced embodied AI, and using them to benchmark robotic intelligence has become a pivotal trend. Implements techniques from the paper 'BiManiBench: A Hierarchical Benchmark for Evaluating Bimanual Coordination of Multimodal Large Language Models' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# BiManiBench: A Hierarchical Benchmark for Evaluating Bimanual Coordination of Multimodal Large Language Models

**Source:** [https://arxiv.org/abs/2602.08392v1](https://arxiv.org/abs/2602.08392v1)
**Category:** cs.RO | **Published:** 2026-02-09 | **Skill Score:** 62
**Authors:** Xin Wu, Zhixuan Liang, Yue Ma...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** bimanibench

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

> Multimodal Large Language Models (MLLMs) have significantly advanced embodied AI, and using them to benchmark robotic intelligence has become a pivotal trend. However, existing frameworks remain predominantly confined to single-arm manipulation, failing to capture the spatio-temporal coordination required for bimanual tasks like lifting a heavy pot. To address this, we introduce BiManiBench, a hierarchical benchmark evaluating MLLMs across three tiers: fundamental spatial reasoning, high-level a

Refer to the [full paper](https://arxiv.org/abs/2602.08392v1) for detailed methodology.