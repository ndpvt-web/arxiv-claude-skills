---
name: "graphseek-nextgeneration-graph-analytics"
description: "Graphs are foundational across domains but remain hard to use without deep expertise. Implements techniques from the paper 'GraphSeek: Next-Generation Graph Analytics with LLMs' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework), (database & query) or when the user references techniques from this research area."
---

# GraphSeek: Next-Generation Graph Analytics with LLMs

**Source:** [https://arxiv.org/abs/2602.11052v1](https://arxiv.org/abs/2602.11052v1)
**Category:** cs.DB | **Published:** 2026-02-11 | **Skill Score:** 82
**Authors:** Maciej Besta, Łukasz Jarmocik, Orest Hrycyna...

## Core Capability

Build and orchestrate AI agent workflows.

## Workflow

1. Decompose complex tasks into subtasks
2. Plan agent coordination and communication patterns
3. Execute subtasks with appropriate tools and models
4. Monitor agent progress and handle failures
5. Aggregate results and present final output

## Agent Coordination

- Decompose complex tasks into independent subtasks where possible
- Use parallel execution for independent subtasks
- Implement error recovery and retry logic
- Aggregate partial results when full completion fails

## Research Context

> Graphs are foundational across domains but remain hard to use without deep expertise. LLMs promise accessible natural language (NL) graph analytics, yet they fail to process industry-scale property graphs effectively and efficiently: such datasets are large, highly heterogeneous, structurally complex, and evolve dynamically. To address this, we devise a novel abstraction for complex multi-query analytics over such graphs. Its key idea is to replace brittle generation of graph queries directly fr

Refer to the [full paper](https://arxiv.org/abs/2602.11052v1) for detailed methodology.