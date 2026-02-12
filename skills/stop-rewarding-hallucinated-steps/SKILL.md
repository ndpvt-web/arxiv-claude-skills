---
name: "stop-rewarding-hallucinated-steps"
description: "As large language models become smaller and more efficient, small reasoning models (SRMs) are crucial for enabling chain-of-thought (CoT) reasoning in resource-constrained settings. Implements techniques from the paper 'Stop Rewarding Hallucinated Steps: Faithfulness-Aware Step-Level Reinforcement Learning for Small Reasoning Models' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework) or when the user references techniques from this research area."
---

# Stop Rewarding Hallucinated Steps: Faithfulness-Aware Step-Level Reinforcement Learning for Small Reasoning Models

**Source:** [https://arxiv.org/abs/2602.05897v1](https://arxiv.org/abs/2602.05897v1)
**Category:** cs.CL | **Published:** 2026-02-05 | **Skill Score:** 64
**Authors:** Shuo Nie, Hexuan Deng, Chao Wang...

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

> As large language models become smaller and more efficient, small reasoning models (SRMs) are crucial for enabling chain-of-thought (CoT) reasoning in resource-constrained settings. However, they are prone to faithfulness hallucinations, especially in intermediate reasoning steps. Existing mitigation methods based on online reinforcement learning rely on outcome-based rewards or coarse-grained CoT evaluation, which can inadvertently reinforce unfaithful reasoning when the final answer is correct

Refer to the [full paper](https://arxiv.org/abs/2602.05897v1) for detailed methodology.