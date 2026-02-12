---
name: "meta-context-engineering-via"
description: "The operational efficacy of large language models relies heavily on their inference-time context. Implements techniques from the paper 'Meta Context Engineering via Agentic Skill Evolution' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework), (database & query) or when the user references techniques from this research area."
---

# Meta Context Engineering via Agentic Skill Evolution

**Source:** [https://arxiv.org/abs/2601.21557v2](https://arxiv.org/abs/2601.21557v2)
**Category:** cs.AI | **Published:** 2026-01-29 | **Skill Score:** 59
**Authors:** Haoran Ye, Xuning He, Vincent Arak...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** meta context engineering (mce)

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

> The operational efficacy of large language models relies heavily on their inference-time context. This has established Context Engineering (CE) as a formal discipline for optimizing these inputs. Current CE methods rely on manually crafted harnesses, such as rigid generation-reflection workflows and predefined context schemas. They impose structural biases and restrict context optimization to a narrow, intuition-bound design space. To address this, we introduce Meta Context Engineering (MCE), a 

Refer to the [full paper](https://arxiv.org/abs/2601.21557v2) for detailed methodology.