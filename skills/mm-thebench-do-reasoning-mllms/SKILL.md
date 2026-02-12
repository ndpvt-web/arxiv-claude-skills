---
name: "mm-thebench-do-reasoning-mllms"
description: "Recent advances in multimodal large language models (MLLMs) mark a shift from non-thinking models to post-trained reasoning models capable of solving complex problems through thinking. Implements techniques from the paper 'MM-THEBench: Do Reasoning MLLMs Think Reasonably?' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework) or when the user references techniques from this research area."
---

# MM-THEBench: Do Reasoning MLLMs Think Reasonably?

**Source:** [https://arxiv.org/abs/2601.22735v1](https://arxiv.org/abs/2601.22735v1)
**Category:** cs.CL | **Published:** 2026-01-30 | **Skill Score:** 59
**Authors:** Zhidian Huang, Zijun Yao, Ji Qi...

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

> Recent advances in multimodal large language models (MLLMs) mark a shift from non-thinking models to post-trained reasoning models capable of solving complex problems through thinking. However, whether such thinking mitigates hallucinations in multimodal perception and reasoning remains unclear. Self-reflective reasoning enhances robustness but introduces additional hallucinations, and subtle perceptual errors still result in incorrect or coincidentally correct answers. Existing benchmarks prima

Refer to the [full paper](https://arxiv.org/abs/2601.22735v1) for detailed methodology.