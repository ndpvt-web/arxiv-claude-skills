---
name: "prism-festina-lente-proactivity"
description: "Proactive agents must decide not only what to say but also whether and when to intervene. Implements techniques from the paper 'PRISM: Festina Lente Proactivity -- Risk-Sensitive, Uncertainty-Aware Deliberation for Proactive Agents' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework), (database & query) or when the user references techniques from this research area."
---

# PRISM: Festina Lente Proactivity -- Risk-Sensitive, Uncertainty-Aware Deliberation for Proactive Agents

**Source:** [https://arxiv.org/abs/2602.01532v1](https://arxiv.org/abs/2602.01532v1)
**Category:** cs.AI | **Published:** 2026-02-02 | **Skill Score:** 60
**Authors:** Yuxuan Fu, Xiaoyu Tan, Teqi Hao...

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

> Proactive agents must decide not only what to say but also whether and when to intervene. Many current systems rely on brittle heuristics or indiscriminate long reasoning, which offers little control over the benefit-burden tradeoff. We formulate the problem as cost-sensitive selective intervention and present PRISM, a novel framework that couples a decision-theoretic gate with a dual-process reasoning architecture. At inference time, the agent intervenes only when a calibrated probability of us

Refer to the [full paper](https://arxiv.org/abs/2602.01532v1) for detailed methodology.