---
name: "optimizing-agentic-workflows-using"
description: "Agentic AI enables LLM to dynamically reason, plan, and interact with tools to solve complex tasks. Implements techniques from the paper 'Optimizing Agentic Workflows using Meta-tools' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework) or when the user references techniques from this research area."
---

# Optimizing Agentic Workflows using Meta-tools

**Source:** [https://arxiv.org/abs/2601.22037v2](https://arxiv.org/abs/2601.22037v2)
**Category:** cs.AI | **Published:** 2026-01-29 | **Skill Score:** 66
**Authors:** Sami Abuzakuk, Anne-Marie Kermarrec, Rishi Sharma...

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

> Agentic AI enables LLM to dynamically reason, plan, and interact with tools to solve complex tasks. However, agentic workflows often require many iterative reasoning steps and tool invocations, leading to significant operational expense, end-to-end latency and failures due to hallucinations. This work introduces Agent Workflow Optimization (AWO), a framework that identifies and optimizes redundant tool execution patterns to improve the efficiency and robustness of agentic workflows. AWO analyzes

Refer to the [full paper](https://arxiv.org/abs/2601.22037v2) for detailed methodology.