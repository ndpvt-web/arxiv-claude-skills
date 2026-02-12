---
name: "drift-bench-diagnosing-cooperative-breakdowns"
description: "As Large Language Models transition to autonomous agents, user inputs frequently violate cooperative assumptions (e.g., implicit intent, missing parameters, false presuppositions, or ambiguous expr... Implements techniques from the paper 'Drift-Bench: Diagnosing Cooperative Breakdowns in LLM Agents under Input Faults via Multi-Turn Interaction' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# Drift-Bench: Diagnosing Cooperative Breakdowns in LLM Agents under Input Faults via Multi-Turn Interaction

**Source:** [https://arxiv.org/abs/2602.02455v1](https://arxiv.org/abs/2602.02455v1)
**Category:** cs.AI | **Published:** 2026-02-02 | **Skill Score:** 91
**Authors:** Han Bao, Zheyuan Zhang, Pengcheng Jing...

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

> As Large Language Models transition to autonomous agents, user inputs frequently violate cooperative assumptions (e.g., implicit intent, missing parameters, false presuppositions, or ambiguous expressions), creating execution risks that text-only evaluations do not capture. Existing benchmarks typically assume well-specified instructions or restrict evaluation to text-only, single-turn clarification, and thus do not measure multi-turn disambiguation under grounded execution risk. We introduce \t

Refer to the [full paper](https://arxiv.org/abs/2602.02455v1) for detailed methodology.