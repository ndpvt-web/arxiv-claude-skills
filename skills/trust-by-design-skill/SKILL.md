---
name: "trust-by-design-skill"
description: "How should Large Language Model (LLM) practitioners select the right model for a task without wasting money? We introduce BELLA (Budget-Efficient LLM Selection via Automated skill-profiling), a fra... Implements techniques from the paper 'Trust by Design: Skill Profiles for Transparent, Cost-Aware LLM Routing' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework) or when the user references techniques from this research area."
---

# Trust by Design: Skill Profiles for Transparent, Cost-Aware LLM Routing

**Source:** [https://arxiv.org/abs/2602.02386v1](https://arxiv.org/abs/2602.02386v1)
**Category:** cs.AI | **Published:** 2026-02-02 | **Skill Score:** 82
**Authors:** Mika Okamoto, Ansel Kaplan Erol, Glenn Matlin

## Core Capability

Build and orchestrate AI agent workflows.

## Key Techniques

- **Proposed technique:** bella (budget-efficient llm selection via automated skill-profiling)

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

> How should Large Language Model (LLM) practitioners select the right model for a task without wasting money? We introduce BELLA (Budget-Efficient LLM Selection via Automated skill-profiling), a framework that recommends optimal LLM selection for tasks through interpretable skill-based model selection. Standard benchmarks report aggregate metrics that obscure which specific capabilities a task requires and whether a cheaper model could suffice. BELLA addresses this gap through three stages: (1) d

Refer to the [full paper](https://arxiv.org/abs/2602.02386v1) for detailed methodology.