---
name: "mtdrive-multiturn-interactive-reinforcement"
description: "Trajectory planning is a core task in autonomous driving, requiring the prediction of safe and comfortable paths across diverse scenarios. Implements techniques from the paper 'MTDrive: Multi-turn Interactive Reinforcement Learning for Autonomous Driving' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework) or when the user references techniques from this research area."
---

# MTDrive: Multi-turn Interactive Reinforcement Learning for Autonomous Driving

**Source:** [https://arxiv.org/abs/2601.22930v1](https://arxiv.org/abs/2601.22930v1)
**Category:** cs.RO | **Published:** 2026-01-30 | **Skill Score:** 58
**Authors:** Xidong Li, Mingyu Guo, Chenchao Xu...

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

> Trajectory planning is a core task in autonomous driving, requiring the prediction of safe and comfortable paths across diverse scenarios. Integrating Multi-modal Large Language Models (MLLMs) with Reinforcement Learning (RL) has shown promise in addressing "long-tail" scenarios. However, existing methods are constrained to single-turn reasoning, limiting their ability to handle complex tasks requiring iterative refinement. To overcome this limitation, we present MTDrive, a multi-turn framework 

Refer to the [full paper](https://arxiv.org/abs/2601.22930v1) for detailed methodology.