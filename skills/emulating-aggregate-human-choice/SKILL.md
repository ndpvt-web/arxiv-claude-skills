---
name: "emulating-aggregate-human-choice"
description: "Cognitive biases often shape human decisions. Implements techniques from the paper 'Emulating Aggregate Human Choice Behavior and Biases with GPT Conversational Agents' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework) or when the user references techniques from this research area."
---

# Emulating Aggregate Human Choice Behavior and Biases with GPT Conversational Agents

**Source:** [https://arxiv.org/abs/2602.05597v1](https://arxiv.org/abs/2602.05597v1)
**Category:** cs.AI | **Published:** 2026-02-05 | **Skill Score:** 64
**Authors:** Stephen Pilli, Vivek Nallur

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

> Cognitive biases often shape human decisions. While large language models (LLMs) have been shown to reproduce well-known biases, a more critical question is whether LLMs can predict biases at the individual level and emulate the dynamics of biased human behavior when contextual factors, such as cognitive load, interact with these biases. We adapted three well-established decision scenarios into a conversational setting and conducted a human experiment (N=1100). Participants engaged with a chatbo

Refer to the [full paper](https://arxiv.org/abs/2602.05597v1) for detailed methodology.