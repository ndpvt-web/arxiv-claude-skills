---
name: "evolving-from-tool-user"
description: "Existing Tool-Integrated Reasoning (TIR) models have effectively extended the question-answering capabilities of LLMs by incorporating external tools. Implements techniques from the paper 'Evolving from Tool User to Creator via Training-Free Experience Reuse in Multimodal Reasoning' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework) or when the user references techniques from this research area."
---

# Evolving from Tool User to Creator via Training-Free Experience Reuse in Multimodal Reasoning

**Source:** [https://arxiv.org/abs/2602.01983v1](https://arxiv.org/abs/2602.01983v1)
**Category:** cs.AI | **Published:** 2026-02-02 | **Skill Score:** 74
**Authors:** Xintian Shen, Jiawei Chen, Lihao Zheng...

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

> Existing Tool-Integrated Reasoning (TIR) models have effectively extended the question-answering capabilities of LLMs by incorporating external tools. However, real-world scenarios present numerous open-ended problems where fixed tools often fail to meet task requirements. Furthermore, the lack of self-optimization mechanisms means that erroneous tool outputs can mislead the LLM's responses. Additionally, the construction of existing tools entails significant manual effort, which consequently co

Refer to the [full paper](https://arxiv.org/abs/2602.01983v1) for detailed methodology.