---
name: "hardware-codesign-scaling-laws"
description: "Vision-Language-Action Models (VLAs) have emerged as a key paradigm of Physical AI and are increasingly deployed in autonomous vehicles, robots, and smart spaces. Implements techniques from the paper 'Hardware Co-Design Scaling Laws via Roofline Modelling for On-Device LLMs' for automate deployment, ci/cd, and infrastructure tasks. Use when tasks involve (devops automation), (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# Hardware Co-Design Scaling Laws via Roofline Modelling for On-Device LLMs

**Source:** [https://arxiv.org/abs/2602.10377v1](https://arxiv.org/abs/2602.10377v1)
**Category:** cs.LG | **Published:** 2026-02-10 | **Skill Score:** 60
**Authors:** Luoyang Sun, Jiwen Jiang, Yifeng Ding...

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

> Vision-Language-Action Models (VLAs) have emerged as a key paradigm of Physical AI and are increasingly deployed in autonomous vehicles, robots, and smart spaces. In these resource-constrained on-device settings, selecting an appropriate large language model (LLM) backbone is a critical challenge: models must balance accuracy with strict inference latency and hardware efficiency constraints. This makes hardware-software co-design a game-changing requirement for on-device LLM deployment, where ea

Refer to the [full paper](https://arxiv.org/abs/2602.10377v1) for detailed methodology.