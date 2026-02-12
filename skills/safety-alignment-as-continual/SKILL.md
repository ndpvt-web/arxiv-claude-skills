---
name: "safety-alignment-as-continual"
description: "Large Language Models (LLMs) often incur an alignment tax: safety post-training can reduce general utility (e.g., reasoning and coding). Implements techniques from the paper 'Safety Alignment as Continual Learning: Mitigating the Alignment Tax via Orthogonal Gradient Projection' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework) or when the user references techniques from this research area."
---

# Safety Alignment as Continual Learning: Mitigating the Alignment Tax via Orthogonal Gradient Projection

**Source:** [https://arxiv.org/abs/2602.07892v1](https://arxiv.org/abs/2602.07892v1)
**Category:** cs.LG | **Published:** 2026-02-08 | **Skill Score:** 72
**Authors:** Guanglong Sun, Siyuan Zhang, Liyuan Wang...

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

> Large Language Models (LLMs) often incur an alignment tax: safety post-training can reduce general utility (e.g., reasoning and coding). We argue that this tax primarily arises from continual-learning-style forgetting in sequential alignment, where distribution shift and conflicting objectives cause safety updates to overwrite pre-trained competencies. Accordingly, we cast safety alignment as a continual learning (CL) problem that must balance plasticity (acquiring safety constraints) and stabil

Refer to the [full paper](https://arxiv.org/abs/2602.07892v1) for detailed methodology.