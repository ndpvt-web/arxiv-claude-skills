---
name: "spell-synthesis-of-programmatic"
description: "Library migration is a common but error-prone task in software development. Implements techniques from the paper 'SPELL: Synthesis of Programmatic Edits using LLMs' for automate deployment, ci/cd, and infrastructure tasks. Use when tasks involve (devops automation), (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# SPELL: Synthesis of Programmatic Edits using LLMs

**Source:** [https://arxiv.org/abs/2602.01107v1](https://arxiv.org/abs/2602.01107v1)
**Category:** cs.SE | **Published:** 2026-02-01 | **Skill Score:** 85
**Authors:** Daniel Ramos, Catarina Gamboa, Inês Lynce...

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

> Library migration is a common but error-prone task in software development. Developers may need to replace one library with another due to reasons like changing requirements or licensing changes. Migration typically entails updating and rewriting source code manually. While automated migration tools exist, most rely on mining examples from real-world projects that have already undergone similar migrations. However, these data are scarce, and collecting them for arbitrary pairs of libraries is di

Refer to the [full paper](https://arxiv.org/abs/2602.01107v1) for detailed methodology.