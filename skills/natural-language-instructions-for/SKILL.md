---
name: "natural-language-instructions-for"
description: "Instruction-grounded driving, where passenger language guides trajectory planning, requires vehicles to understand intent before motion. Implements techniques from the paper 'Natural Language Instructions for Scene-Responsive Human-in-the-Loop Motion Planning in Autonomous Driving using Vision-Language-Action Models' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework), (prompt engineering), (design & ui) or when the user references techniques from this research area."
---

# Natural Language Instructions for Scene-Responsive Human-in-the-Loop Motion Planning in Autonomous Driving using Vision-Language-Action Models

**Source:** [https://arxiv.org/abs/2602.04184v1](https://arxiv.org/abs/2602.04184v1)
**Category:** cs.CV | **Published:** 2026-02-04 | **Skill Score:** 75
**Authors:** Angel Martinez-Sanchez, Parthib Roy, Ross Greer

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

> Instruction-grounded driving, where passenger language guides trajectory planning, requires vehicles to understand intent before motion. However, most prior instruction-following planners rely on simulation or fixed command vocabularies, limiting real-world generalization. doScenes, the first real-world dataset linking free-form instructions (with referentiality) to nuScenes ground-truth motion, enables instruction-conditioned planning. In this work, we adapt OpenEMMA, an open-source MLLM-based 

Refer to the [full paper](https://arxiv.org/abs/2602.04184v1) for detailed methodology.