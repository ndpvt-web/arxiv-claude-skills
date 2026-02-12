---
name: "task-oriented-robothuman-handovers-on"
description: "Task-oriented handovers (TOH) are fundamental to effective human-robot collaboration, requiring robots to present objects in a way that supports the human's intended post-handover use. Implements techniques from the paper 'Task-Oriented Robot-Human Handovers on Legged Manipulators' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework), (prompt engineering), (database & query) or when the user references techniques from this research area."
---

# Task-Oriented Robot-Human Handovers on Legged Manipulators

**Source:** [https://arxiv.org/abs/2602.05760v1](https://arxiv.org/abs/2602.05760v1)
**Category:** cs.RO | **Published:** 2026-02-05 | **Skill Score:** 81
**Authors:** Andreea Tulbure, Carmen Scheidemann, Elias Steiner...

## Core Capability

Build and orchestrate AI agent workflows.

## Key Techniques

- **Proposed technique:** aft-handover

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

> Task-oriented handovers (TOH) are fundamental to effective human-robot collaboration, requiring robots to present objects in a way that supports the human's intended post-handover use. Existing approaches are typically based on object- or task-specific affordances, but their ability to generalize to novel scenarios is limited. To address this gap, we present AFT-Handover, a framework that integrates large language model (LLM)-driven affordance reasoning with efficient texture-based affordance tr

Refer to the [full paper](https://arxiv.org/abs/2602.05760v1) for detailed methodology.