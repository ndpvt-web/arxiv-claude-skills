---
name: "tracecoder-a-tracedriven-multiagent"
description: "Large Language Models (LLMs) often generate code with subtle but critical bugs, especially for complex tasks. Implements techniques from the paper 'TraceCoder: A Trace-Driven Multi-Agent Framework for Automated Debugging of LLM-Generated Code' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework), (design & ui) or when the user references techniques from this research area."
---

# TraceCoder: A Trace-Driven Multi-Agent Framework for Automated Debugging of LLM-Generated Code

**Source:** [https://arxiv.org/abs/2602.06875v1](https://arxiv.org/abs/2602.06875v1)
**Category:** cs.SE | **Published:** 2026-02-06 | **Skill Score:** 78
**Authors:** Jiangping Huang, Wenguang Ye, Weisong Sun...

## Core Capability

Build and orchestrate AI agent workflows.

## Key Techniques

- **Multi-agent architecture** for task decomposition and parallel execution

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

> Large Language Models (LLMs) often generate code with subtle but critical bugs, especially for complex tasks. Existing automated repair methods typically rely on superficial pass/fail signals, offering limited visibility into program behavior and hindering precise error localization. In addition, without a way to learn from prior failures, repair processes often fall into repetitive and inefficient cycles. To overcome these challenges, we present TraceCoder, a collaborative multi-agent framework

Refer to the [full paper](https://arxiv.org/abs/2602.06875v1) for detailed methodology.