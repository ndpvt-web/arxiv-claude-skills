---
name: "when-single-answer-is"
description: "Recent progress has expanded the use of large language models (LLMs) in drug discovery, including synthesis planning. Implements techniques from the paper 'When Single Answer Is Not Enough: Rethinking Single-Step Retrosynthesis Benchmarks for LLMs' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework) or when the user references techniques from this research area."
---

# When Single Answer Is Not Enough: Rethinking Single-Step Retrosynthesis Benchmarks for LLMs

**Source:** [https://arxiv.org/abs/2602.03554v1](https://arxiv.org/abs/2602.03554v1)
**Category:** cs.LG | **Published:** 2026-02-03 | **Skill Score:** 64
**Authors:** Bogdan Zagribelnyy, Ivan Ilin, Maksim Kuznetsov...

## Core Capability

Build and orchestrate AI agent workflows.

## Key Techniques

- **Proposed technique:** a new benchmarking framework for single-step retrosynthesis that evaluates both gener
- **Novel approach:** benchmarking framework

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

> Recent progress has expanded the use of large language models (LLMs) in drug discovery, including synthesis planning. However, objective evaluation of retrosynthesis performance remains limited. Existing benchmarks and metrics typically rely on published synthetic procedures and Top-K accuracy based on single ground-truth, which does not capture the open-ended nature of real-world synthesis planning. We propose a new benchmarking framework for single-step retrosynthesis that evaluates both gener

Refer to the [full paper](https://arxiv.org/abs/2602.03554v1) for detailed methodology.