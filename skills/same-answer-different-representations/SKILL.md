---
name: "same-answer-different-representations"
description: "The robustness of Vision Language Models (VLMs) is commonly assessed through output-level invariance, implicitly assuming that stable predictions reflect stable multimodal processing. Implements techniques from the paper 'Same Answer, Different Representations: Hidden instability in VLMs' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# Same Answer, Different Representations: Hidden instability in VLMs

**Source:** [https://arxiv.org/abs/2602.06652v1](https://arxiv.org/abs/2602.06652v1)
**Category:** cs.AI | **Published:** 2026-02-06 | **Skill Score:** 62
**Authors:** Farooq Ahmad Wani, Alessandro Suglia, Rohit Saxena...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** a representation-aware and frequency-aware evaluation framework that measures internal embedding drift

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

> The robustness of Vision Language Models (VLMs) is commonly assessed through output-level invariance, implicitly assuming that stable predictions reflect stable multimodal processing. In this work, we argue that this assumption is insufficient. We introduce a representation-aware and frequency-aware evaluation framework that measures internal embedding drift, spectral sensitivity, and structural smoothness (spatial consistency of vision tokens), alongside standard label-based metrics. Applying t

Refer to the [full paper](https://arxiv.org/abs/2602.06652v1) for detailed methodology.