---
name: "hippasus-effective-and-efficient"
description: "Machine learning models depend critically on feature quality, yet useful features are often scattered across multiple relational tables. Implements techniques from the paper 'Hippasus: Effective and Efficient Automatic Feature Augmentation for Machine Learning Tasks on Relational Data' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework), (database & query) or when the user references techniques from this research area."
---

# Hippasus: Effective and Efficient Automatic Feature Augmentation for Machine Learning Tasks on Relational Data

**Source:** [https://arxiv.org/abs/2602.02025v1](https://arxiv.org/abs/2602.02025v1)
**Category:** cs.DB | **Published:** 2026-02-02 | **Skill Score:** 64
**Authors:** Serafeim Papadias, Kostas Patroumpas, Dimitrios Skoutas

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

> Machine learning models depend critically on feature quality, yet useful features are often scattered across multiple relational tables. Feature augmentation enriches a base table by discovering and integrating features from related tables through join operations. However, scaling this process to complex schemas with many tables and multi-hop paths remains challenging. Feature augmentation must address three core tasks: identify promising join paths that connect the base table to candidate table

Refer to the [full paper](https://arxiv.org/abs/2602.02025v1) for detailed methodology.