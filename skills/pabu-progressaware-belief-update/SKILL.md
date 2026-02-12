---
name: "pabu-progressaware-belief-update"
description: "Large Language Model (LLM) agents commonly condition actions on full action-observation histories, which introduce task-irrelevant information that easily leads to redundant actions and higher infe... Implements techniques from the paper 'PABU: Progress-Aware Belief Update for Efficient LLM Agents' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# PABU: Progress-Aware Belief Update for Efficient LLM Agents

**Source:** [https://arxiv.org/abs/2602.09138v1](https://arxiv.org/abs/2602.09138v1)
**Category:** cs.AI | **Published:** 2026-02-09 | **Skill Score:** 68
**Authors:** Haitao Jiang, Lin Ge, Hengrui Cai...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** progress-aware belief update (pabu)

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

> Large Language Model (LLM) agents commonly condition actions on full action-observation histories, which introduce task-irrelevant information that easily leads to redundant actions and higher inference cost. We propose Progress-Aware Belief Update (PABU), a belief-state framework that compactly represents an agent's state by explicitly modeling task progress and selectively retaining past actions and observations. At each step, the agent predicts its relative progress since the previous round a

Refer to the [full paper](https://arxiv.org/abs/2602.09138v1) for detailed methodology.