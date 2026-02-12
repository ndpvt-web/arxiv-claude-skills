---
name: "livemedbench-a-contaminationfree-medical"
description: "The deployment of Large Language Models (LLMs) in high-stakes clinical settings demands rigorous and reliable evaluation. Implements techniques from the paper 'LiveMedBench: A Contamination-Free Medical Benchmark for LLMs with Automated Rubric Evaluation' for automate deployment, ci/cd, and infrastructure tasks. Use when tasks involve (devops automation), (agent framework) or when the user references techniques from this research area."
---

# LiveMedBench: A Contamination-Free Medical Benchmark for LLMs with Automated Rubric Evaluation

**Source:** [https://arxiv.org/abs/2602.10367v1](https://arxiv.org/abs/2602.10367v1)
**Category:** cs.AI | **Published:** 2026-02-10 | **Skill Score:** 69
**Authors:** Zhiling Yan, Dingjie Song, Zhe Fang...

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

> The deployment of Large Language Models (LLMs) in high-stakes clinical settings demands rigorous and reliable evaluation. However, existing medical benchmarks remain static, suffering from two critical limitations: (1) data contamination, where test sets inadvertently leak into training corpora, leading to inflated performance estimates; and (2) temporal misalignment, failing to capture the rapid evolution of medical knowledge. Furthermore, current evaluation metrics for open-ended clinical reas

Refer to the [full paper](https://arxiv.org/abs/2602.10367v1) for detailed methodology.