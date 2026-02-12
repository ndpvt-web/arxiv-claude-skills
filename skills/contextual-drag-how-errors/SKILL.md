---
name: "contextual-drag-how-errors"
description: "Central to many self-improvement pipelines for large language models (LLMs) is the assumption that models can improve by reflecting on past mistakes. Implements techniques from the paper 'Contextual Drag: How Errors in the Context Affect LLM Reasoning' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# Contextual Drag: How Errors in the Context Affect LLM Reasoning

**Source:** [https://arxiv.org/abs/2602.04288v1](https://arxiv.org/abs/2602.04288v1)
**Category:** cs.CL | **Published:** 2026-02-04 | **Skill Score:** 74
**Authors:** Yun Cheng, Xingyu Zhu, Haoyu Zhao...

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

> Central to many self-improvement pipelines for large language models (LLMs) is the assumption that models can improve by reflecting on past mistakes. We study a phenomenon termed contextual drag: the presence of failed attempts in the context biases subsequent generations toward structurally similar errors. Across evaluations of 11 proprietary and open-weight models on 8 reasoning tasks, contextual drag induces 10-20% performance drops, and iterative self-refinement in models with severe context

Refer to the [full paper](https://arxiv.org/abs/2602.04288v1) for detailed methodology.