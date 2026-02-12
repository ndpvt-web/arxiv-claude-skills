---
name: "internalizing-llm-reasoning-via"
description: "The internalization of chain-of-thought processes into hidden states has emerged as a highly efficient paradigm for scaling test-time compute. Implements techniques from the paper 'Internalizing LLM Reasoning via Discovery and Replay of Latent Actions' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# Internalizing LLM Reasoning via Discovery and Replay of Latent Actions

**Source:** [https://arxiv.org/abs/2602.04925v1](https://arxiv.org/abs/2602.04925v1)
**Category:** cs.LG | **Published:** 2026-02-04 | **Skill Score:** 89
**Authors:** Zhenning Shi, Yijia Zhu, Junhan Shi...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** stir (self-distilled tools for internal reasoning)

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

> The internalization of chain-of-thought processes into hidden states has emerged as a highly efficient paradigm for scaling test-time compute. However, existing activation steering methods rely on static control vectors that fail to adapt to the non-stationary evolution of complex reasoning tasks. To address this limitation, we propose STIR (Self-Distilled Tools for Internal Reasoning), a framework that reformulates reasoning enhancement as a dynamic latent trajectory control problem. STIR intro

Refer to the [full paper](https://arxiv.org/abs/2602.04925v1) for detailed methodology.