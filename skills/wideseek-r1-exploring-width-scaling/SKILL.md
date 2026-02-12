---
name: "wideseek-r1-exploring-width-scaling"
description: "Recent advancements in Large Language Models (LLMs) have largely focused on depth scaling, where a single agent solves long-horizon problems with multi-turn reasoning and tool use. Implements techniques from the paper 'WideSeek-R1: Exploring Width Scaling for Broad Information Seeking via Multi-Agent Reinforcement Learning' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# WideSeek-R1: Exploring Width Scaling for Broad Information Seeking via Multi-Agent Reinforcement Learning

**Source:** [https://arxiv.org/abs/2602.04634v1](https://arxiv.org/abs/2602.04634v1)
**Category:** cs.AI | **Published:** 2026-02-04 | **Skill Score:** 93
**Authors:** Zelai Xu, Zhexuan Xu, Ruize Zhang...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Multi-agent architecture** for task decomposition and parallel execution

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

> Recent advancements in Large Language Models (LLMs) have largely focused on depth scaling, where a single agent solves long-horizon problems with multi-turn reasoning and tool use. However, as tasks grow broader, the key bottleneck shifts from individual competence to organizational capability. In this work, we explore a complementary dimension of width scaling with multi-agent systems to address broad information seeking. Existing multi-agent systems often rely on hand-crafted workflows and tur

Refer to the [full paper](https://arxiv.org/abs/2602.04634v1) for detailed methodology.