---
name: "to-think-or-not"
description: "Theory of Mind (ToM) assesses whether models can infer hidden mental states such as beliefs, desires, and intentions, which is essential for natural social interaction. Implements techniques from the paper 'To Think or Not To Think, That is The Question for Large Reasoning Models in Theory of Mind Tasks' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework) or when the user references techniques from this research area."
---

# To Think or Not To Think, That is The Question for Large Reasoning Models in Theory of Mind Tasks

**Source:** [https://arxiv.org/abs/2602.10625v1](https://arxiv.org/abs/2602.10625v1)
**Category:** cs.AI | **Published:** 2026-02-11 | **Skill Score:** 64
**Authors:** Nanxu Gong, Haotian Li, Sixun Dong...

## Core Capability

Build and orchestrate AI agent workflows.

## Key Techniques

- **Proposed technique:** a systematic study of nine advanced large language models (llms)

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

> Theory of Mind (ToM) assesses whether models can infer hidden mental states such as beliefs, desires, and intentions, which is essential for natural social interaction. Although recent progress in Large Reasoning Models (LRMs) has boosted step-by-step inference in mathematics and coding, it is still underexplored whether this benefit transfers to socio-cognitive skills. We present a systematic study of nine advanced Large Language Models (LLMs), comparing reasoning models with non-reasoning mode

Refer to the [full paper](https://arxiv.org/abs/2602.10625v1) for detailed methodology.