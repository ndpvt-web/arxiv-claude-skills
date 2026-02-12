---
name: "training-data-selection-with"
description: "Fine-tuning large language models (LLMs) for specialized domains often necessitates a trade-off between acquiring domain expertise and retaining general reasoning capabilities, a phenomenon known a... Implements techniques from the paper 'Training Data Selection with Gradient Orthogonality for Efficient Domain Adaptation' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# Training Data Selection with Gradient Orthogonality for Efficient Domain Adaptation

**Source:** [https://arxiv.org/abs/2602.06359v1](https://arxiv.org/abs/2602.06359v1)
**Category:** cs.LG | **Published:** 2026-02-06 | **Skill Score:** 66
**Authors:** Xiyang Zhang, Yuanhe Tian, Hongzhi Wang...

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

> Fine-tuning large language models (LLMs) for specialized domains often necessitates a trade-off between acquiring domain expertise and retaining general reasoning capabilities, a phenomenon known as catastrophic forgetting. Existing remedies face a dichotomy: gradient surgery methods offer geometric safety but incur prohibitive computational costs via online projections, while efficient data selection approaches reduce overhead but remain blind to conflict-inducing gradient directions. In this p

Refer to the [full paper](https://arxiv.org/abs/2602.06359v1) for detailed methodology.