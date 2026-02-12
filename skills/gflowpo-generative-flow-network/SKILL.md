---
name: "gflowpo-generative-flow-network"
description: "Finding effective prompts for language models (LMs) is critical yet notoriously difficult: the prompt space is combinatorially large, rewards are sparse due to expensive target-LM evaluation. Implements techniques from the paper 'GFlowPO: Generative Flow Network as a Language Model Prompt Optimizer' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (prompt engineering) or when the user references techniques from this research area."
---

# GFlowPO: Generative Flow Network as a Language Model Prompt Optimizer

**Source:** [https://arxiv.org/abs/2602.03358v1](https://arxiv.org/abs/2602.03358v1)
**Category:** cs.AI | **Published:** 2026-02-03 | **Skill Score:** 75
**Authors:** Junmo Cho, Suhan Kim, Sangjune An...

## Core Capability

Search, retrieve, and synthesize information.

## Workflow

1. Parse the user's information need or query
2. Formulate effective search strategies
3. Retrieve relevant documents, code, or data
4. Synthesize findings into a coherent response
5. Provide citations and references

## Research Context

> Finding effective prompts for language models (LMs) is critical yet notoriously difficult: the prompt space is combinatorially large, rewards are sparse due to expensive target-LM evaluation. Yet, existing RL-based prompt optimizers often rely on on-policy updates and a meta-prompt sampled from a fixed distribution, leading to poor sample efficiency. We propose GFlowPO, a probabilistic prompt optimization framework that casts prompt search as a posterior inference problem over latent prompts reg

Refer to the [full paper](https://arxiv.org/abs/2602.03358v1) for detailed methodology.