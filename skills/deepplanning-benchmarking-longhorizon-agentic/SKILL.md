---
name: "deepplanning-benchmarking-longhorizon-agentic"
description: "While agent evaluation has shifted toward long-horizon tasks, most benchmarks still emphasize local, step-level reasoning rather than the global constrained optimization (e.g., time and financial b... Implements techniques from the paper 'DeepPlanning: Benchmarking Long-Horizon Agentic Planning with Verifiable Constraints' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# DeepPlanning: Benchmarking Long-Horizon Agentic Planning with Verifiable Constraints

**Source:** [https://arxiv.org/abs/2601.18137v1](https://arxiv.org/abs/2601.18137v1)
**Category:** cs.AI | **Published:** 2026-01-26 | **Skill Score:** 80
**Authors:** Yinger Zhang, Shutong Jiang, Renhao Li...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** deepplanning

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

> While agent evaluation has shifted toward long-horizon tasks, most benchmarks still emphasize local, step-level reasoning rather than the global constrained optimization (e.g., time and financial budgets) that demands genuine planning ability. Meanwhile, existing LLM planning benchmarks underrepresent the active information gathering and fine-grained local constraints typical of real-world settings. To address this, we introduce DeepPlanning, a challenging benchmark for practical long-horizon ag

Refer to the [full paper](https://arxiv.org/abs/2601.18137v1) for detailed methodology.