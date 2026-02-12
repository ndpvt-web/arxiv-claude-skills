---
name: "swe-pruner-selfadaptive-context-pruning"
description: "LLM agents have demonstrated remarkable capabilities in software development, but their performance is hampered by long interaction contexts, which incur high API costs and latency. Implements techniques from the paper 'SWE-Pruner: Self-Adaptive Context Pruning for Coding Agents' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework) or when the user references techniques from this research area."
---

# SWE-Pruner: Self-Adaptive Context Pruning for Coding Agents

**Source:** [https://arxiv.org/abs/2601.16746v2](https://arxiv.org/abs/2601.16746v2)
**Category:** cs.SE | **Published:** 2026-01-23 | **Skill Score:** 71
**Authors:** Yuhang Wang, Yuling Shi, Mo Yang...

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

> LLM agents have demonstrated remarkable capabilities in software development, but their performance is hampered by long interaction contexts, which incur high API costs and latency. While various context compression approaches such as LongLLMLingua have emerged to tackle this challenge, they typically rely on fixed metrics such as PPL, ignoring the task-specific nature of code understanding. As a result, they frequently disrupt syntactic and logical structure and fail to retain critical implemen

Refer to the [full paper](https://arxiv.org/abs/2601.16746v2) for detailed methodology.