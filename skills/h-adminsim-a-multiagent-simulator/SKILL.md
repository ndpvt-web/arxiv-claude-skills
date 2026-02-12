---
name: "h-adminsim-a-multiagent-simulator"
description: "Hospital administration departments handle a wide range of operational tasks and, in large hospitals, process over 10,000 requests per day, driving growing interest in LLM-based automation. Implements techniques from the paper 'H-AdminSim: A Multi-Agent Simulator for Realistic Hospital Administrative Workflows with FHIR Integration' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework) or when the user references techniques from this research area."
---

# H-AdminSim: A Multi-Agent Simulator for Realistic Hospital Administrative Workflows with FHIR Integration

**Source:** [https://arxiv.org/abs/2602.05407v1](https://arxiv.org/abs/2602.05407v1)
**Category:** cs.AI | **Published:** 2026-02-05 | **Skill Score:** 64
**Authors:** Jun-Min Lee, Meong Hi Son, Edward Choi

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

> Hospital administration departments handle a wide range of operational tasks and, in large hospitals, process over 10,000 requests per day, driving growing interest in LLM-based automation. However, prior work has focused primarily on patient--physician interactions or isolated administrative subtasks, failing to capture the complexity of real administrative workflows. To address this gap, we propose H-AdminSim, a comprehensive end-to-end simulation framework that combines realistic data generat

Refer to the [full paper](https://arxiv.org/abs/2602.05407v1) for detailed methodology.