---
name: "thinking-makes-llm-agents"
description: "Eliciting reasoning has emerged as a powerful technique for improving the performance of large language models (LLMs) on complex tasks by inducing thinking. Implements techniques from the paper 'Thinking Makes LLM Agents Introverted: How Mandatory Thinking Can Backfire in User-Engaged Agents' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# Thinking Makes LLM Agents Introverted: How Mandatory Thinking Can Backfire in User-Engaged Agents

**Source:** [https://arxiv.org/abs/2602.07796v1](https://arxiv.org/abs/2602.07796v1)
**Category:** cs.CL | **Published:** 2026-02-08 | **Skill Score:** 71
**Authors:** Jiatong Li, Changdae Oh, Hyeong Kyu Choi...

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

> Eliciting reasoning has emerged as a powerful technique for improving the performance of large language models (LLMs) on complex tasks by inducing thinking. However, their effectiveness in realistic user-engaged agent scenarios remains unclear. In this paper, we conduct a comprehensive study on the effect of explicit thinking in user-engaged LLM agents. Our experiments span across seven models, three benchmarks, and two thinking instantiations, and we evaluate them through both a quantitative re

Refer to the [full paper](https://arxiv.org/abs/2602.07796v1) for detailed methodology.