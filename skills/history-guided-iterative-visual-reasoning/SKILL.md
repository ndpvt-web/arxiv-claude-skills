---
name: "history-guided-iterative-visual-reasoning"
description: "Self-consistency methods are the core technique for improving the reasoning reliability of multimodal large language models (MLLMs). Implements techniques from the paper 'History-Guided Iterative Visual Reasoning with Self-Correction' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# History-Guided Iterative Visual Reasoning with Self-Correction

**Source:** [https://arxiv.org/abs/2602.04413v1](https://arxiv.org/abs/2602.04413v1)
**Category:** cs.CL | **Published:** 2026-02-04 | **Skill Score:** 67
**Authors:** Xinglong Yang, Zhilin Peng, Zhanzhan Liu...

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

> Self-consistency methods are the core technique for improving the reasoning reliability of multimodal large language models (MLLMs). By generating multiple reasoning results through repeated sampling and selecting the best answer via voting, they play an important role in cross-modal tasks. However, most existing self-consistency methods are limited to a fixed ``repeated sampling and voting'' paradigm and do not reuse historical reasoning information. As a result, models struggle to actively cor

Refer to the [full paper](https://arxiv.org/abs/2602.04413v1) for detailed methodology.