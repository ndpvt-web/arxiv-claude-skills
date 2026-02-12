---
name: "when-and-how-much"
description: "Despite rapid progress in Multimodal Large Language Models (MLLMs), visual spatial reasoning remains unreliable when correct answers depend on how a scene would appear under unseen or alternative v... Implements techniques from the paper 'When and How Much to Imagine: Adaptive Test-Time Scaling with World Models for Visual Spatial Reasoning' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework) or when the user references techniques from this research area."
---

# When and How Much to Imagine: Adaptive Test-Time Scaling with World Models for Visual Spatial Reasoning

**Source:** [https://arxiv.org/abs/2602.08236v1](https://arxiv.org/abs/2602.08236v1)
**Category:** cs.CV | **Published:** 2026-02-09 | **Skill Score:** 70
**Authors:** Shoubin Yu, Yue Zhang, Zun Wang...

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

> Despite rapid progress in Multimodal Large Language Models (MLLMs), visual spatial reasoning remains unreliable when correct answers depend on how a scene would appear under unseen or alternative viewpoints. Recent work addresses this by augmenting reasoning with world models for visual imagination, but questions such as when imagination is actually necessary, how much of it is beneficial, and when it becomes harmful, remain poorly understood. In practice, indiscriminate imagination can increase

Refer to the [full paper](https://arxiv.org/abs/2602.08236v1) for detailed methodology.