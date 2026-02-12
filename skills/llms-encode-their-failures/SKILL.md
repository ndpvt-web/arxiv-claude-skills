---
name: "llms-encode-their-failures"
description: "Running LLMs with extended reasoning on every problem is expensive, but determining which inputs actually require additional compute remains challenging. Implements techniques from the paper 'LLMs Encode Their Failures: Predicting Success from Pre-Generation Activations' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# LLMs Encode Their Failures: Predicting Success from Pre-Generation Activations

**Source:** [https://arxiv.org/abs/2602.09924v1](https://arxiv.org/abs/2602.09924v1)
**Category:** cs.CL | **Published:** 2026-02-10 | **Skill Score:** 76
**Authors:** William Lugoloobi, Thomas Foster, William Bankes...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Achievement:** surface features such as

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

> Running LLMs with extended reasoning on every problem is expensive, but determining which inputs actually require additional compute remains challenging. We investigate whether their own likelihood of success is recoverable from their internal representations before generation, and if this signal can guide more efficient inference. We train linear probes on pre-generation activations to predict policy-specific success on math and coding tasks, substantially outperforming surface features such as

Refer to the [full paper](https://arxiv.org/abs/2602.09924v1) for detailed methodology.