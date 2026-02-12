---
name: "evermembench-benchmarking-longterm-interactive"
description: "Long-term conversational memory is essential for LLM-based assistants, yet existing benchmarks focus on dyadic, single-topic dialogues that fail to capture real-world complexity. Implements techniques from the paper 'EverMemBench: Benchmarking Long-Term Interactive Memory in Large Language Models' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# EverMemBench: Benchmarking Long-Term Interactive Memory in Large Language Models

**Source:** [https://arxiv.org/abs/2602.01313v2](https://arxiv.org/abs/2602.01313v2)
**Category:** cs.CL | **Published:** 2026-02-01 | **Skill Score:** 69
**Authors:** Chuanrui Hu, Tong Li, Xingze Gao...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** evermembench

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

> Long-term conversational memory is essential for LLM-based assistants, yet existing benchmarks focus on dyadic, single-topic dialogues that fail to capture real-world complexity. We introduce EverMemBench, a benchmark featuring multi-party, multi-group conversations spanning over 1 million tokens with temporally evolving information, cross-topic interleaving, and role-specific personas. EverMemBench evaluates memory systems across three dimensions through 1,000+ QA pairs: fine-grained recall, me

Refer to the [full paper](https://arxiv.org/abs/2602.01313v2) for detailed methodology.