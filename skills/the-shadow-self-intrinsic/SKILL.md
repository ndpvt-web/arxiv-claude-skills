---
name: "the-shadow-self-intrinsic"
description: "Large language model (LLM) agents with extended autonomy unlock new capabilities, but also introduce heightened challenges for LLM safety. Implements techniques from the paper 'The Shadow Self: Intrinsic Value Misalignment in Large Language Model Agents' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# The Shadow Self: Intrinsic Value Misalignment in Large Language Model Agents

**Source:** [https://arxiv.org/abs/2601.17344v1](https://arxiv.org/abs/2601.17344v1)
**Category:** cs.CL | **Published:** 2026-01-24 | **Skill Score:** 76
**Authors:** Chen Chen, Kim Young Il, Yuan Yang...

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

> Large language model (LLM) agents with extended autonomy unlock new capabilities, but also introduce heightened challenges for LLM safety. In particular, an LLM agent may pursue objectives that deviate from human values and ethical norms, a risk known as value misalignment. Existing evaluations primarily focus on responses to explicit harmful input or robustness against system failure, while value misalignment in realistic, fully benign, and agentic settings remains largely underexplored. To fil

Refer to the [full paper](https://arxiv.org/abs/2601.17344v1) for detailed methodology.