---
name: "refer-agent-a-collaborative-multiagent"
description: "Referring Video Object Segmentation (RVOS) aims to segment objects in videos based on textual queries. Implements techniques from the paper 'Refer-Agent: A Collaborative Multi-Agent System with Reasoning and Reflection for Referring Video Object Segmentation' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework), (prompt engineering), (design & ui) or when the user references techniques from this research area."
---

# Refer-Agent: A Collaborative Multi-Agent System with Reasoning and Reflection for Referring Video Object Segmentation

**Source:** [https://arxiv.org/abs/2602.03595v2](https://arxiv.org/abs/2602.03595v2)
**Category:** cs.CV | **Published:** 2026-02-03 | **Skill Score:** 83
**Authors:** Haichao Jiang, Tianming Liang, Wei-Shi Zheng...

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

> Referring Video Object Segmentation (RVOS) aims to segment objects in videos based on textual queries. Current methods mainly rely on large-scale supervised fine-tuning (SFT) of Multi-modal Large Language Models (MLLMs). However, this paradigm suffers from heavy data dependence and limited scalability against the rapid evolution of MLLMs. Although recent zero-shot approaches offer a flexible alternative, their performance remains significantly behind SFT-based methods, due to the straightforward

Refer to the [full paper](https://arxiv.org/abs/2602.03595v2) for detailed methodology.