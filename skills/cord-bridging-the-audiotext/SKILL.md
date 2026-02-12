---
name: "cord-bridging-the-audiotext"
description: "Large Audio Language Models (LALMs) have garnered significant research interest. Implements techniques from the paper 'CORD: Bridging the Audio-Text Reasoning Gap via Weighted On-policy Cross-modal Distillation' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (content generation), (agent framework) or when the user references techniques from this research area."
---

# CORD: Bridging the Audio-Text Reasoning Gap via Weighted On-policy Cross-modal Distillation

**Source:** [https://arxiv.org/abs/2601.16547v1](https://arxiv.org/abs/2601.16547v1)
**Category:** cs.SD | **Published:** 2026-01-23 | **Skill Score:** 68
**Authors:** Jing Hu, Danxiang Zhu, Xianlong Luo...

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

> Large Audio Language Models (LALMs) have garnered significant research interest. Despite being built upon text-based large language models (LLMs), LALMs frequently exhibit a degradation in knowledge and reasoning capabilities. We hypothesize that this limitation stems from the failure of current training paradigms to effectively bridge the acoustic-semantic gap within the feature representation space. To address this challenge, we propose CORD, a unified alignment framework that performs online 

Refer to the [full paper](https://arxiv.org/abs/2601.16547v1) for detailed methodology.