---
name: "learning-queryaware-budgettier-routing"
description: "Memory is increasingly central to Large Language Model (LLM) agents operating beyond a single context window, yet most existing systems rely on offline, query-agnostic memory construction that can ... Implements techniques from the paper 'Learning Query-Aware Budget-Tier Routing for Runtime Agent Memory' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework) or when the user references techniques from this research area."
---

# Learning Query-Aware Budget-Tier Routing for Runtime Agent Memory

**Source:** [https://arxiv.org/abs/2602.06025v1](https://arxiv.org/abs/2602.06025v1)
**Category:** cs.CL | **Published:** 2026-02-05 | **Skill Score:** 71
**Authors:** Haozhen Zhang, Haodong Yue, Tao Feng...

## Core Capability

Build and orchestrate AI agent workflows.

## Key Techniques

- **Proposed technique:** \textbf{budgetmem}

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

> Memory is increasingly central to Large Language Model (LLM) agents operating beyond a single context window, yet most existing systems rely on offline, query-agnostic memory construction that can be inefficient and may discard query-critical information. Although runtime memory utilization is a natural alternative, prior work often incurs substantial overhead and offers limited explicit control over the performance-cost trade-off. In this work, we present \textbf{BudgetMem}, a runtime agent mem

Refer to the [full paper](https://arxiv.org/abs/2602.06025v1) for detailed methodology.