---
name: "refuge-feature-generation-for"
description: "Relational databases (RDBs) play a crucial role in many real-world web applications, supporting data management across multiple interconnected tables. Implements techniques from the paper 'ReFuGe: Feature Generation for Prediction Tasks on Relational Databases with LLM Agents' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework), (database & query) or when the user references techniques from this research area."
---

# ReFuGe: Feature Generation for Prediction Tasks on Relational Databases with LLM Agents

**Source:** [https://arxiv.org/abs/2601.17735v1](https://arxiv.org/abs/2601.17735v1)
**Category:** cs.AI | **Published:** 2026-01-25 | **Skill Score:** 80
**Authors:** Kyungho Kim, Geon Lee, Juyeon Kim...

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

> Relational databases (RDBs) play a crucial role in many real-world web applications, supporting data management across multiple interconnected tables. Beyond typical retrieval-oriented tasks, prediction tasks on RDBs have recently gained attention. In this work, we address this problem by generating informative relational features that enhance predictive performance. However, generating such features is challenging: it requires reasoning over complex schemas and exploring a combinatorially large

Refer to the [full paper](https://arxiv.org/abs/2601.17735v1) for detailed methodology.