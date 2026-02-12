---
name: "drpg-decompose-retrieve-plan"
description: "Despite the growing adoption of large language models (LLMs) in scientific research workflows, automated support for academic rebuttal, a crucial step in academic communication and peer review, rem... Implements techniques from the paper 'DRPG (Decompose, Retrieve, Plan, Generate): An Agentic Framework for Academic Rebuttal' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# DRPG (Decompose, Retrieve, Plan, Generate): An Agentic Framework for Academic Rebuttal

**Source:** [https://arxiv.org/abs/2601.18081v1](https://arxiv.org/abs/2601.18081v1)
**Category:** cs.LG | **Published:** 2026-01-26 | **Skill Score:** 74
**Authors:** Peixuan Han, Yingjie Yu, Jingjun Xu...

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

> Despite the growing adoption of large language models (LLMs) in scientific research workflows, automated support for academic rebuttal, a crucial step in academic communication and peer review, remains largely underexplored. Existing approaches typically rely on off-the-shelf LLMs or simple pipelines, which struggle with long-context understanding and often fail to produce targeted and persuasive responses. In this paper, we propose DRPG, an agentic framework for automatic academic rebuttal gene

Refer to the [full paper](https://arxiv.org/abs/2601.18081v1) for detailed methodology.