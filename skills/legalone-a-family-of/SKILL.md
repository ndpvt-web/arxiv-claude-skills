---
name: "legalone-a-family-of"
description: "While Large Language Models (LLMs) have demonstrated impressive general capabilities, their direct application in the legal domain is often hindered by a lack of precise domain knowledge and comple... Implements techniques from the paper 'LegalOne: A Family of Foundation Models for Reliable Legal Reasoning' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework) or when the user references techniques from this research area."
---

# LegalOne: A Family of Foundation Models for Reliable Legal Reasoning

**Source:** [https://arxiv.org/abs/2602.00642v2](https://arxiv.org/abs/2602.00642v2)
**Category:** cs.CL | **Published:** 2026-01-31 | **Skill Score:** 81
**Authors:** Haitao Li, Yifan Chen, Shuo Miao...

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

> While Large Language Models (LLMs) have demonstrated impressive general capabilities, their direct application in the legal domain is often hindered by a lack of precise domain knowledge and complexity of performing rigorous multi-step judicial reasoning. To address this gap, we present LegalOne, a family of foundational models specifically tailored for the Chinese legal domain. LegalOne is developed through a comprehensive three-phase pipeline designed to master legal reasoning. First, during m

Refer to the [full paper](https://arxiv.org/abs/2602.00642v2) for detailed methodology.