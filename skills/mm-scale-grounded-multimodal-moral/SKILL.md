---
name: "mm-scale-grounded-multimodal-moral"
description: "Vision-Language Models (VLMs) continue to struggle to make morally salient judgments in multimodal and socially ambiguous contexts. Implements techniques from the paper 'MM-SCALE: Grounded Multimodal Moral Reasoning via Scalar Judgment and Listwise Alignment' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework) or when the user references techniques from this research area."
---

# MM-SCALE: Grounded Multimodal Moral Reasoning via Scalar Judgment and Listwise Alignment

**Source:** [https://arxiv.org/abs/2602.03665v1](https://arxiv.org/abs/2602.03665v1)
**Category:** cs.CV | **Published:** 2026-02-03 | **Skill Score:** 59
**Authors:** Eunkyu Park, Wesley Hanwen Deng, Cheyon Jin...

## Core Capability

Build and orchestrate AI agent workflows.

## Key Techniques

- **Proposed technique:** mm-scale (multimodal moral scale)

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

> Vision-Language Models (VLMs) continue to struggle to make morally salient judgments in multimodal and socially ambiguous contexts. Prior works typically rely on binary or pairwise supervision, which often fail to capture the continuous and pluralistic nature of human moral reasoning. We present MM-SCALE (Multimodal Moral Scale), a large-scale dataset for aligning VLMs with human moral preferences through 5-point scalar ratings and explicit modality grounding. Each image-scenario pair is annotat

Refer to the [full paper](https://arxiv.org/abs/2602.03665v1) for detailed methodology.