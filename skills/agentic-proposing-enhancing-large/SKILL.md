---
name: "agentic-proposing-enhancing-large"
description: "Advancing complex reasoning in large language models relies on high-quality, verifiable datasets, yet human annotation remains cost-prohibitive and difficult to scale. Implements techniques from the paper 'Agentic Proposing: Enhancing Large Language Model Reasoning via Compositional Skill Synthesis' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework) or when the user references techniques from this research area."
---

# Agentic Proposing: Enhancing Large Language Model Reasoning via Compositional Skill Synthesis

**Source:** [https://arxiv.org/abs/2602.03279v1](https://arxiv.org/abs/2602.03279v1)
**Category:** cs.AI | **Published:** 2026-02-03 | **Skill Score:** 58
**Authors:** Zhengbo Jiao, Shaobo Wang, Zifan Zhang...

## Core Capability

Build and orchestrate AI agent workflows.

## Key Techniques

- **Proposed technique:** agentic proposing

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

> Advancing complex reasoning in large language models relies on high-quality, verifiable datasets, yet human annotation remains cost-prohibitive and difficult to scale. Current synthesis paradigms often face a recurring trade-off: maintaining structural validity typically restricts problem complexity, while relaxing constraints to increase difficulty frequently leads to inconsistent or unsolvable instances. To address this, we propose Agentic Proposing, a framework that models problem synthesis a

Refer to the [full paper](https://arxiv.org/abs/2602.03279v1) for detailed methodology.