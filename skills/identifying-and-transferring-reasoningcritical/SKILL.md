---
name: "identifying-and-transferring-reasoningcritical"
description: "Despite the strong reasoning capabilities of recent large language models (LLMs), achieving reliable performance on challenging tasks often requires post-training or computationally expensive sampl... Implements techniques from the paper 'Identifying and Transferring Reasoning-Critical Neurons: Improving LLM Inference Reliability via Activation Steering' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework) or when the user references techniques from this research area."
---

# Identifying and Transferring Reasoning-Critical Neurons: Improving LLM Inference Reliability via Activation Steering

**Source:** [https://arxiv.org/abs/2601.19847v1](https://arxiv.org/abs/2601.19847v1)
**Category:** cs.CL | **Published:** 2026-01-27 | **Skill Score:** 58
**Authors:** Fangan Dong, Zuming Yan, Xuri Ge...

## Core Capability

Build and orchestrate AI agent workflows.

## Key Techniques

- **Proposed technique:** adaras (adaptive reasoning activation steering)
- **Achievement:** reliable performance on challenging tasks often requires post-training or computationally expensive sampling strategies

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

> Despite the strong reasoning capabilities of recent large language models (LLMs), achieving reliable performance on challenging tasks often requires post-training or computationally expensive sampling strategies, limiting their practical efficiency. In this work, we first show that a small subset of neurons in LLMs exhibits strong predictive correlations with reasoning correctness. Based on this observation, we propose AdaRAS (Adaptive Reasoning Activation Steering), a lightweight test-time fram

Refer to the [full paper](https://arxiv.org/abs/2601.19847v1) for detailed methodology.