---
name: "who-deserves-the-reward"
description: "Integrating Large Language Models (LLMs) with external tools via multi-agent systems offers a promising new paradigm for decomposing and solving complex problems. Implements techniques from the paper 'Who Deserves the Reward? SHARP: Shapley Credit-based Optimization for Multi-Agent System' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# Who Deserves the Reward? SHARP: Shapley Credit-based Optimization for Multi-Agent System

**Source:** [https://arxiv.org/abs/2602.08335v1](https://arxiv.org/abs/2602.08335v1)
**Category:** cs.AI | **Published:** 2026-02-09 | **Skill Score:** 66
**Authors:** Yanming Li, Xuelin Zhang, WenJie Lu...

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

> Integrating Large Language Models (LLMs) with external tools via multi-agent systems offers a promising new paradigm for decomposing and solving complex problems. However, training these systems remains notoriously difficult due to the credit assignment challenge, as it is often unclear which specific functional agent is responsible for the success or failure of decision trajectories. Existing methods typically rely on sparse or globally broadcast rewards, failing to capture individual contribut

Refer to the [full paper](https://arxiv.org/abs/2602.08335v1) for detailed methodology.