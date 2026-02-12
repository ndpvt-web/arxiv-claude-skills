---
name: "agent-skills-a-datadriven"
description: "Agent skills extend large language model (LLM) agents with reusable, program-like modules that define triggering conditions, procedural logic, and tool interactions. Implements techniques from the paper 'Agent Skills: A Data-Driven Analysis of Claude Skills for Extending Large Language Model Functionality' for automate deployment, ci/cd, and infrastructure tasks. Use when tasks involve (devops automation), (search & retrieval), (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# Agent Skills: A Data-Driven Analysis of Claude Skills for Extending Large Language Model Functionality

**Source:** [https://arxiv.org/abs/2602.08004v1](https://arxiv.org/abs/2602.08004v1)
**Category:** cs.SE | **Published:** 2026-02-08 | **Skill Score:** 61
**Authors:** George Ling, Shanshan Zhong, Richard Huang

## Core Capability

Automate deployment, CI/CD, and infrastructure tasks.

## Workflow

1. Analyze the current infrastructure and deployment setup
2. Design automation workflows for the target environment
3. Generate configuration files (Docker, K8s, CI/CD pipelines)
4. Implement monitoring and alerting
5. Validate the automation works correctly

## Agent Coordination

- Decompose complex tasks into independent subtasks where possible
- Use parallel execution for independent subtasks
- Implement error recovery and retry logic
- Aggregate partial results when full completion fails

## Research Context

> Agent skills extend large language model (LLM) agents with reusable, program-like modules that define triggering conditions, procedural logic, and tool interactions. As these skills proliferate in public marketplaces, it is unclear what types are available, how users adopt them, and what risks they pose. To answer these questions, we conduct a large-scale, data-driven analysis of 40,285 publicly listed skills from a major marketplace. Our results show that skill publication tends to occur in sho

Refer to the [full paper](https://arxiv.org/abs/2602.08004v1) for detailed methodology.