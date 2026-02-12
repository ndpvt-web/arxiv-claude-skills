---
name: "locomo-plus-beyondfactual-cognitive-memory"
description: "Long-term conversational memory is a core capability for LLM-based dialogue systems, yet existing benchmarks and evaluation protocols primarily focus on surface-level factual recall. Implements techniques from the paper 'Locomo-Plus: Beyond-Factual Cognitive Memory Evaluation Framework for LLM Agents' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# Locomo-Plus: Beyond-Factual Cognitive Memory Evaluation Framework for LLM Agents

**Source:** [https://arxiv.org/abs/2602.10715v1](https://arxiv.org/abs/2602.10715v1)
**Category:** cs.CL | **Published:** 2026-02-11 | **Skill Score:** 83
**Authors:** Yifei Li, Weidong Guo, Lingling Zhang...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** \textbf{locomo-plus}

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

> Long-term conversational memory is a core capability for LLM-based dialogue systems, yet existing benchmarks and evaluation protocols primarily focus on surface-level factual recall. In realistic interactions, appropriate responses often depend on implicit constraints such as user state, goals, or values that are not explicitly queried later. To evaluate this setting, we introduce \textbf{LoCoMo-Plus}, a benchmark for assessing cognitive memory under cue--trigger semantic disconnect, where model

Refer to the [full paper](https://arxiv.org/abs/2602.10715v1) for detailed methodology.