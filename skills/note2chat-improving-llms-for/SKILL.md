---
name: "note2chat-improving-llms-for"
description: "Effective clinical history taking is a foundational yet underexplored component of clinical reasoning. Implements techniques from the paper 'Note2Chat: Improving LLMs for Multi-Turn Clinical History Taking Using Medical Notes' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework), (design & ui) or when the user references techniques from this research area."
---

# Note2Chat: Improving LLMs for Multi-Turn Clinical History Taking Using Medical Notes

**Source:** [https://arxiv.org/abs/2601.21551v1](https://arxiv.org/abs/2601.21551v1)
**Category:** cs.CL | **Published:** 2026-01-29 | **Skill Score:** 80
**Authors:** Yang Zhou, Zhenting Sheng, Mingrui Tan...

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

> Effective clinical history taking is a foundational yet underexplored component of clinical reasoning. While large language models (LLMs) have shown promise on static benchmarks, they often fall short in dynamic, multi-turn diagnostic settings that require iterative questioning and hypothesis refinement. To address this gap, we propose \method{}, a note-driven framework that trains LLMs to conduct structured history taking and diagnosis by learning from widely available medical notes. Instead of

Refer to the [full paper](https://arxiv.org/abs/2601.21551v1) for detailed methodology.