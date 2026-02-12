---
name: "aster-agentic-scaling-with"
description: "Reinforcement learning (RL) has emerged as a dominant paradigm for eliciting long-horizon reasoning in Large Language Models (LLMs). Implements techniques from the paper 'ASTER: Agentic Scaling with Tool-integrated Extended Reasoning' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework) or when the user references techniques from this research area."
---

# ASTER: Agentic Scaling with Tool-integrated Extended Reasoning

**Source:** [https://arxiv.org/abs/2602.01204v1](https://arxiv.org/abs/2602.01204v1)
**Category:** cs.CL | **Published:** 2026-02-01 | **Skill Score:** 63
**Authors:** Xuqin Zhang, Quan He, Zhenrui Zheng...

## Core Capability

Build and orchestrate AI agent workflows.

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

> Reinforcement learning (RL) has emerged as a dominant paradigm for eliciting long-horizon reasoning in Large Language Models (LLMs). However, scaling Tool-Integrated Reasoning (TIR) via RL remains challenging due to interaction collapse: a pathological state where models fail to sustain multi-turn tool usage, instead degenerating into heavy internal reasoning with only trivial, post-hoc code verification. We systematically study three questions: (i) how cold-start SFT induces an agentic, tool-us

Refer to the [full paper](https://arxiv.org/abs/2602.01204v1) for detailed methodology.