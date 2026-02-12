---
name: "from-answer-givers-to"
description: "Design feedback helps practitioners improve their artifacts while also fostering reflection and design reasoning. Implements techniques from the paper 'From Answer Givers to Design Mentors: Guiding LLMs with the Cognitive Apprenticeship Model' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# From Answer Givers to Design Mentors: Guiding LLMs with the Cognitive Apprenticeship Model

**Source:** [https://arxiv.org/abs/2601.19053v1](https://arxiv.org/abs/2601.19053v1)
**Category:** cs.HC | **Published:** 2026-01-27 | **Skill Score:** 62
**Authors:** Yongsu Ahn, Lejun R Liao, Benjamin Bach...

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

> Design feedback helps practitioners improve their artifacts while also fostering reflection and design reasoning. Large Language Models (LLMs) such as ChatGPT can support design work, but often provide generic, one-off suggestions that limit reflective engagement. We investigate how to guide LLMs to act as design mentors by applying the Cognitive Apprenticeship Model, which emphasizes demonstrating reasoning through six methods: modeling, coaching, scaffolding, articulation, reflection, and expl

Refer to the [full paper](https://arxiv.org/abs/2601.19053v1) for detailed methodology.