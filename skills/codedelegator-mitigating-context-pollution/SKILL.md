---
name: "codedelegator-mitigating-context-pollution"
description: "Recent advances in large language models (LLMs) allow agents to represent actions as executable code, offering greater expressivity than traditional tool-calling. Implements techniques from the paper 'CodeDelegator: Mitigating Context Pollution via Role Separation in Code-as-Action Agents' for automate deployment, ci/cd, and infrastructure tasks. Use when tasks involve (devops automation), (agent framework) or when the user references techniques from this research area."
---

# CodeDelegator: Mitigating Context Pollution via Role Separation in Code-as-Action Agents

**Source:** [https://arxiv.org/abs/2601.14914v1](https://arxiv.org/abs/2601.14914v1)
**Category:** cs.CL | **Published:** 2026-01-21 | **Skill Score:** 64
**Authors:** Tianxiang Fei, Cheng Chen, Yue Pan...

## Core Capability

Automate deployment, CI/CD, and infrastructure tasks.

## Key Techniques

- **Proposed technique:** codedelegator
- **Multi-agent architecture** for task decomposition and parallel execution

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

> Recent advances in large language models (LLMs) allow agents to represent actions as executable code, offering greater expressivity than traditional tool-calling. However, real-world tasks often demand both strategic planning and detailed implementation. Using a single agent for both leads to context pollution from debugging traces and intermediate failures, impairing long-horizon performance. We propose CodeDelegator, a multi-agent framework that separates planning from implementation via role 

Refer to the [full paper](https://arxiv.org/abs/2601.14914v1) for detailed methodology.