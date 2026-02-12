---
name: "self-improving-world-modelling-with"
description: "Internal modelling of the world -- predicting transitions between previous states $X$ and next states $Y$ under actions $Z$ -- is essential to reasoning and planning for LLMs and VLMs. Implements techniques from the paper 'Self-Improving World Modelling with Latent Actions' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# Self-Improving World Modelling with Latent Actions

**Source:** [https://arxiv.org/abs/2602.06130v1](https://arxiv.org/abs/2602.06130v1)
**Category:** cs.LG | **Published:** 2026-02-05 | **Skill Score:** 68
**Authors:** Yifu Qiu, Zheng Zhao, Waylon Li...

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

> Internal modelling of the world -- predicting transitions between previous states $X$ and next states $Y$ under actions $Z$ -- is essential to reasoning and planning for LLMs and VLMs. Learning such models typically requires costly action-labelled trajectories. We propose SWIRL, a self-improvement framework that learns from state-only sequences by treating actions as a latent variable and alternating between Forward World Modelling (FWM) $P_θ(Y|X,Z)$ and an Inverse Dynamics Modelling (IDM) $Q_φ(

Refer to the [full paper](https://arxiv.org/abs/2602.06130v1) for detailed methodology.