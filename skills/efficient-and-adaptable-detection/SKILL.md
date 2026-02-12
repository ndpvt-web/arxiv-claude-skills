---
name: "efficient-and-adaptable-detection"
description: "Large Language Models (LLMs) have demonstrated remarkable capabilities in natural language understanding, reasoning, and generation. Implements techniques from the paper 'Efficient and Adaptable Detection of Malicious LLM Prompts via Bootstrap Aggregation' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# Efficient and Adaptable Detection of Malicious LLM Prompts via Bootstrap Aggregation

**Source:** [https://arxiv.org/abs/2602.08062v1](https://arxiv.org/abs/2602.08062v1)
**Category:** cs.LG | **Published:** 2026-02-08 | **Skill Score:** 69
**Authors:** Shayan Ali Hassan, Tao Ni, Zafar Ayyub Qazi...

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

> Large Language Models (LLMs) have demonstrated remarkable capabilities in natural language understanding, reasoning, and generation. However, these systems remain susceptible to malicious prompts that induce unsafe or policy-violating behavior through harmful requests, jailbreak techniques, and prompt injection attacks. Existing defenses face fundamental limitations: black-box moderation APIs offer limited transparency and adapt poorly to evolving threats, while white-box approaches using large 

Refer to the [full paper](https://arxiv.org/abs/2602.08062v1) for detailed methodology.