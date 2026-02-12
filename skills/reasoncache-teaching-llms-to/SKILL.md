---
name: "reasoncache-teaching-llms-to"
description: "Can Large language models (LLMs) learn to reason without any weight update and only through in-context learning (ICL)? ICL is strikingly sample-efficient, often learning from only a handful of demo... Implements techniques from the paper 'ReasonCACHE: Teaching LLMs To Reason Without Weight Updates' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# ReasonCACHE: Teaching LLMs To Reason Without Weight Updates

**Source:** [https://arxiv.org/abs/2602.02366v1](https://arxiv.org/abs/2602.02366v1)
**Category:** cs.LG | **Published:** 2026-02-02 | **Skill Score:** 62
**Authors:** Sharut Gupta, Phillip Isola, Stefanie Jegelka...

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

> Can Large language models (LLMs) learn to reason without any weight update and only through in-context learning (ICL)? ICL is strikingly sample-efficient, often learning from only a handful of demonstrations, but complex reasoning tasks typically demand many training examples to learn from. However, naively scaling ICL by adding more demonstrations breaks down at this scale: attention costs grow quadratically, performance saturates or degrades with longer contexts, and the approach remains a sha

Refer to the [full paper](https://arxiv.org/abs/2602.02366v1) for detailed methodology.