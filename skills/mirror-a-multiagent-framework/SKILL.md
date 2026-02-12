---
name: "mirror-a-multiagent-framework"
description: "Operations Research (OR) relies on expert-driven modeling-a slow and fragile process ill-suited to novel scenarios. Implements techniques from the paper 'MIRROR: A Multi-Agent Framework with Iterative Adaptive Revision and Hierarchical Retrieval for Optimization Modeling in Operations Research' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# MIRROR: A Multi-Agent Framework with Iterative Adaptive Revision and Hierarchical Retrieval for Optimization Modeling in Operations Research

**Source:** [https://arxiv.org/abs/2602.03318v2](https://arxiv.org/abs/2602.03318v2)
**Category:** cs.CL | **Published:** 2026-02-03 | **Skill Score:** 76
**Authors:** Yifan Shi, Jialong Shi, Jiayi Wang...

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

> Operations Research (OR) relies on expert-driven modeling-a slow and fragile process ill-suited to novel scenarios. While large language models (LLMs) can automatically translate natural language into optimization models, existing approaches either rely on costly post-training or employ multi-agent frameworks, yet most still lack reliable collaborative error correction and task-specific retrieval, often leading to incorrect outputs. We propose MIRROR, a fine-tuning-free, end-to-end multi-agent f

Refer to the [full paper](https://arxiv.org/abs/2602.03318v2) for detailed methodology.