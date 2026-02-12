---
name: "fine-r1-make-multimodal-llms"
description: "Any entity in the visual world can be hierarchically grouped based on shared characteristics and mapped to fine-grained sub-categories. Implements techniques from the paper 'Fine-R1: Make Multi-modal LLMs Excel in Fine-Grained Visual Recognition by Chain-of-Thought Reasoning' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework) or when the user references techniques from this research area."
---

# Fine-R1: Make Multi-modal LLMs Excel in Fine-Grained Visual Recognition by Chain-of-Thought Reasoning

**Source:** [https://arxiv.org/abs/2602.07605v2](https://arxiv.org/abs/2602.07605v2)
**Category:** cs.CV | **Published:** 2026-02-07 | **Skill Score:** 77
**Authors:** Hulingxiao He, Zijun Geng, Yuxin Peng

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

> Any entity in the visual world can be hierarchically grouped based on shared characteristics and mapped to fine-grained sub-categories. While Multi-modal Large Language Models (MLLMs) achieve strong performance on coarse-grained visual tasks, they often struggle with Fine-Grained Visual Recognition (FGVR). Adapting general-purpose MLLMs to FGVR typically requires large amounts of annotated data, which is costly to obtain, leaving a substantial performance gap compared to contrastive CLIP models 

Refer to the [full paper](https://arxiv.org/abs/2602.07605v2) for detailed methodology.