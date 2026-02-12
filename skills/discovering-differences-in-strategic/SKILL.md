---
name: "discovering-differences-in-strategic"
description: "As Large Language Models (LLMs) are increasingly deployed in social and strategic scenarios, it becomes critical to understand where and why their behavior diverges from that of humans. Implements techniques from the paper 'Discovering Differences in Strategic Behavior Between Humans and LLMs' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework) or when the user references techniques from this research area."
---

# Discovering Differences in Strategic Behavior Between Humans and LLMs

**Source:** [https://arxiv.org/abs/2602.10324v1](https://arxiv.org/abs/2602.10324v1)
**Category:** cs.AI | **Published:** 2026-02-10 | **Skill Score:** 63
**Authors:** Caroline Wang, Daniel Kasenberg, Kim Stachenfeld...

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

> As Large Language Models (LLMs) are increasingly deployed in social and strategic scenarios, it becomes critical to understand where and why their behavior diverges from that of humans. While behavioral game theory (BGT) provides a framework for analyzing behavior, existing models do not fully capture the idiosyncratic behavior of humans or black-box, non-human agents like LLMs. We employ AlphaEvolve, a cutting-edge program discovery tool, to directly discover interpretable models of human and L

Refer to the [full paper](https://arxiv.org/abs/2602.10324v1) for detailed methodology.