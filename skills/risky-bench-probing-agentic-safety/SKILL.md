---
name: "risky-bench-probing-agentic-safety"
description: "Large Language Models (LLMs) are increasingly deployed as agents that operate in real-world environments, introducing safety risks beyond linguistic harm. Implements techniques from the paper 'Risky-Bench: Probing Agentic Safety Risks under Real-World Deployment' for automate deployment, ci/cd, and infrastructure tasks. Use when tasks involve (devops automation), (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# Risky-Bench: Probing Agentic Safety Risks under Real-World Deployment

**Source:** [https://arxiv.org/abs/2602.03100v1](https://arxiv.org/abs/2602.03100v1)
**Category:** cs.AI | **Published:** 2026-02-03 | **Skill Score:** 64
**Authors:** Jingnan Zheng, Yanzhen Luo, Jingjun Xu...

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

> Large Language Models (LLMs) are increasingly deployed as agents that operate in real-world environments, introducing safety risks beyond linguistic harm. Existing agent safety evaluations rely on risk-oriented tasks tailored to specific agent settings, resulting in limited coverage of safety risk space and failing to assess agent safety behavior during long-horizon, interactive task execution in complex real-world deployments. Moreover, their specialization to particular agent settings limits a

Refer to the [full paper](https://arxiv.org/abs/2602.03100v1) for detailed methodology.