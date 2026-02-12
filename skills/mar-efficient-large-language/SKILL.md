---
name: "mar-efficient-large-language"
description: "Large Language Models (LLMs) excel across diverse domains but suffer from high energy costs due to quadratic attention and dense Feed-Forward Network (FFN) operations. Implements techniques from the paper 'MAR: Efficient Large Language Models via Module-aware Architecture Refinement' for build and orchestrate ai agent workflows. Use when tasks involve (general AI assistance) or when the user references techniques from this research area."
---

# MAR: Efficient Large Language Models via Module-aware Architecture Refinement

**Source:** [https://arxiv.org/abs/2601.21503v1](https://arxiv.org/abs/2601.21503v1)
**Category:** cs.AI | **Published:** 2026-01-29 | **Skill Score:** 63
**Authors:** Junhong Cai, Guiqin Wang, Kejie Zhao...

## Core Capability

Build and orchestrate AI agent workflows.

## Key Techniques

- **Proposed technique:** module-aware architecture refinement (mar)

## Workflow

1. Decompose complex tasks into subtasks
2. Plan agent coordination and communication patterns
3. Execute subtasks with appropriate tools and models
4. Monitor agent progress and handle failures
5. Aggregate results and present final output

## Research Context

> Large Language Models (LLMs) excel across diverse domains but suffer from high energy costs due to quadratic attention and dense Feed-Forward Network (FFN) operations. To address these issues, we propose Module-aware Architecture Refinement (MAR), a two-stage framework that integrates State Space Models (SSMs) for linear-time sequence modeling and applies activation sparsification to reduce FFN costs. In addition, to mitigate low information density and temporal mismatch in integrating Spiking N

Refer to the [full paper](https://arxiv.org/abs/2601.21503v1) for detailed methodology.