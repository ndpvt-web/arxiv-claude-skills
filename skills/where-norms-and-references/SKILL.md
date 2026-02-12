---
name: "where-norms-and-references"
description: "Embodied agents, such as robots, will need to interact in situated environments where successful communication often depends on reasoning over social norms: shared expectations that constrain what ... Implements techniques from the paper 'Where Norms and References Collide: Evaluating LLMs on Normative Reasoning' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework) or when the user references techniques from this research area."
---

# Where Norms and References Collide: Evaluating LLMs on Normative Reasoning

**Source:** [https://arxiv.org/abs/2602.02975v1](https://arxiv.org/abs/2602.02975v1)
**Category:** cs.CL | **Published:** 2026-02-03 | **Skill Score:** 63
**Authors:** Mitchell Abrams, Kaveh Eskandari Miandoab, Felix Gervits...

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

> Embodied agents, such as robots, will need to interact in situated environments where successful communication often depends on reasoning over social norms: shared expectations that constrain what actions are appropriate in context. A key capability in such settings is norm-based reference resolution (NBRR), where interpreting referential expressions requires inferring implicit normative expectations grounded in physical and social context. Yet it remains unclear whether Large Language Models (L

Refer to the [full paper](https://arxiv.org/abs/2602.02975v1) for detailed methodology.