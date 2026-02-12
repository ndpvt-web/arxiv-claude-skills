---
name: "chainrec-an-agentic-recommender"
description: "Large language models (LLMs) are increasingly integrated into recommender systems, motivating recent interest in agentic and reasoning-based recommendation. Implements techniques from the paper 'ChainRec: An Agentic Recommender Learning to Route Tool Chains for Diverse and Evolving Interests' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework) or when the user references techniques from this research area."
---

# ChainRec: An Agentic Recommender Learning to Route Tool Chains for Diverse and Evolving Interests

**Source:** [https://arxiv.org/abs/2602.10490v1](https://arxiv.org/abs/2602.10490v1)
**Category:** cs.IR | **Published:** 2026-02-11 | **Skill Score:** 61
**Authors:** Fuchun Li, Qian Li, Xingyu Gao...

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

> Large language models (LLMs) are increasingly integrated into recommender systems, motivating recent interest in agentic and reasoning-based recommendation. However, most existing approaches still rely on fixed workflows, applying the same reasoning procedure across diverse recommendation scenarios. In practice, user contexts vary substantially-for example, in cold-start settings or during interest shifts, so an agent should adaptively decide what evidence to gather next rather than following a 

Refer to the [full paper](https://arxiv.org/abs/2602.10490v1) for detailed methodology.