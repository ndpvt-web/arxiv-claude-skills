---
name: "graphdancer-training-llms-to"
description: "Large language models (LLMs) increasingly rely on external knowledge to improve factuality, yet many real-world knowledge sources are organized as heterogeneous graphs rather than plain text. Implements techniques from the paper 'GraphDancer: Training LLMs to Explore and Reason over Graphs via Curriculum Reinforcement Learning' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework), (database & query) or when the user references techniques from this research area."
---

# GraphDancer: Training LLMs to Explore and Reason over Graphs via Curriculum Reinforcement Learning

**Source:** [https://arxiv.org/abs/2602.02518v1](https://arxiv.org/abs/2602.02518v1)
**Category:** cs.LG | **Published:** 2026-01-24 | **Skill Score:** 70
**Authors:** Yuyang Bai, Zhuofeng Li, Ping Nie...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Retrieval-augmented** approach for grounding responses in external knowledge

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

> Large language models (LLMs) increasingly rely on external knowledge to improve factuality, yet many real-world knowledge sources are organized as heterogeneous graphs rather than plain text. Reasoning over such graph-structured knowledge poses two key challenges: (1) navigating structured, schema-defined relations requires precise function calls rather than similarity-based retrieval, and (2) answering complex questions often demands multi-hop evidence aggregation through iterative information 

Refer to the [full paper](https://arxiv.org/abs/2602.02518v1) for detailed methodology.