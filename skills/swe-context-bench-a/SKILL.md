---
name: "swe-context-bench-a"
description: "Large language models are increasingly used as programming agents for repository level software engineering tasks. Implements techniques from the paper 'SWE Context Bench: A Benchmark for Context Learning in Coding' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# SWE Context Bench: A Benchmark for Context Learning in Coding

**Source:** [https://arxiv.org/abs/2602.08316v1](https://arxiv.org/abs/2602.08316v1)
**Category:** cs.SE | **Published:** 2026-02-09 | **Skill Score:** 66
**Authors:** Jared Zhu, Minhao Hu, Junde Wu

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** swe-contextbench

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

> Large language models are increasingly used as programming agents for repository level software engineering tasks. While recent benchmarks evaluate correctness in realistic codebases, they largely treat tasks as independent and do not assess whether agents can reuse experience across related problems. As a result, the ability of agents to accumulate, retrieve, and apply prior experience, as well as the efficiency gains from such reuse, remains difficult to measure. We introduce SWE-ContextBench,

Refer to the [full paper](https://arxiv.org/abs/2602.08316v1) for detailed methodology.