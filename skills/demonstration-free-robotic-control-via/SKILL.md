---
name: "demonstration-free-robotic-control-via"
description: "Robotic manipulation has increasingly adopted vision-language-action (VLA) models, which achieve strong performance but typically require task-specific demonstrations and fine-tuning, and often gen... Implements techniques from the paper 'Demonstration-Free Robotic Control via LLM Agents' for automate deployment, ci/cd, and infrastructure tasks. Use when tasks involve (devops automation), (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# Demonstration-Free Robotic Control via LLM Agents

**Source:** [https://arxiv.org/abs/2601.20334v1](https://arxiv.org/abs/2601.20334v1)
**Category:** cs.RO | **Published:** 2026-01-28 | **Skill Score:** 91
**Authors:** Brian Y. Tsui, Alan Y. Fang, Tiffany J. Hwu

## Core Capability

Automate deployment, CI/CD, and infrastructure tasks.

## Key Techniques

- **Proposed technique:** faea (frontier agent as embodied agent)

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

> Robotic manipulation has increasingly adopted vision-language-action (VLA) models, which achieve strong performance but typically require task-specific demonstrations and fine-tuning, and often generalize poorly under domain shift. We investigate whether general-purpose large language model (LLM) agent frameworks, originally developed for software engineering, can serve as an alternative control paradigm for embodied manipulation. We introduce FAEA (Frontier Agent as Embodied Agent), which appli

Refer to the [full paper](https://arxiv.org/abs/2601.20334v1) for detailed methodology.