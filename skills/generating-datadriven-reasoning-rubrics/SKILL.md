---
name: "generating-datadriven-reasoning-rubrics"
description: "An impediment to using Large Language Models (LLMs) for reasoning output verification is that LLMs struggle to reliably identify errors in thinking traces, particularly in long outputs, domains req... Implements techniques from the paper 'Generating Data-Driven Reasoning Rubrics for Domain-Adaptive Reward Modeling' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# Generating Data-Driven Reasoning Rubrics for Domain-Adaptive Reward Modeling

**Source:** [https://arxiv.org/abs/2602.06795v1](https://arxiv.org/abs/2602.06795v1)
**Category:** cs.CL | **Published:** 2026-02-06 | **Skill Score:** 67
**Authors:** Kate Sanders, Nathaniel Weir, Sapana Chaudhary...

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

> An impediment to using Large Language Models (LLMs) for reasoning output verification is that LLMs struggle to reliably identify errors in thinking traces, particularly in long outputs, domains requiring expert knowledge, and problems without verifiable rewards. We propose a data-driven approach to automatically construct highly granular reasoning error taxonomies to enhance LLM-driven error detection on unseen reasoning traces. Our findings indicate that classification approaches that leverage 

Refer to the [full paper](https://arxiv.org/abs/2602.06795v1) for detailed methodology.