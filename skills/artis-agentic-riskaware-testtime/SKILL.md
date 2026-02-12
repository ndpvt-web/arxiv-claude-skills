---
name: "artis-agentic-riskaware-testtime"
description: "Current test-time scaling (TTS) techniques enhance large language model (LLM) performance by allocating additional computation at inference time, yet they remain insufficient for agentic settings, ... Implements techniques from the paper 'ARTIS: Agentic Risk-Aware Test-Time Scaling via Iterative Simulation' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework) or when the user references techniques from this research area."
---

# ARTIS: Agentic Risk-Aware Test-Time Scaling via Iterative Simulation

**Source:** [https://arxiv.org/abs/2602.01709v2](https://arxiv.org/abs/2602.01709v2)
**Category:** cs.CL | **Published:** 2026-02-02 | **Skill Score:** 59
**Authors:** Xingshan Zeng, Lingzhi Wang, Weiwen Liu...

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

> Current test-time scaling (TTS) techniques enhance large language model (LLM) performance by allocating additional computation at inference time, yet they remain insufficient for agentic settings, where actions directly interact with external environments and their effects can be irreversible and costly. We propose ARTIS, Agentic Risk-Aware Test-Time Scaling via Iterative Simulation, a framework that decouples exploration from commitment by enabling test-time exploration through simulated intera

Refer to the [full paper](https://arxiv.org/abs/2602.01709v2) for detailed methodology.