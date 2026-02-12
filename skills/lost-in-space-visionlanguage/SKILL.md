---
name: "lost-in-space-visionlanguage"
description: "Vision-Language Models (VLMs) perform well in 2D perception and semantic reasoning compared to their limited understanding of 3D spatial structure. Implements techniques from the paper 'Lost in Space? Vision-Language Models Struggle with Relative Camera Pose Estimation' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework) or when the user references techniques from this research area."
---

# Lost in Space? Vision-Language Models Struggle with Relative Camera Pose Estimation

**Source:** [https://arxiv.org/abs/2601.22228v1](https://arxiv.org/abs/2601.22228v1)
**Category:** cs.CV | **Published:** 2026-01-29 | **Skill Score:** 58
**Authors:** Ken Deng, Yifu Qiu, Yoni Kasten...

## Core Capability

Build and orchestrate AI agent workflows.

## Key Techniques

- **Proposed technique:** vrrpi-bench

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

> Vision-Language Models (VLMs) perform well in 2D perception and semantic reasoning compared to their limited understanding of 3D spatial structure. We investigate this gap using relative camera pose estimation (RCPE), a fundamental vision task that requires inferring relative camera translation and rotation from a pair of images. We introduce VRRPI-Bench, a benchmark derived from unlabeled egocentric videos with verbalized annotations of relative camera motion, reflecting realistic scenarios wit

Refer to the [full paper](https://arxiv.org/abs/2601.22228v1) for detailed methodology.