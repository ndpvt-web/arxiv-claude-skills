---
name: "star-similarityguided-teacherassisted-refinement"
description: "The proliferation of Large Language Models (LLMs) in function calling is pivotal for creating advanced AI agents, yet their large scale hinders widespread adoption, necessitating transferring their... Implements techniques from the paper 'STAR: Similarity-guided Teacher-Assisted Refinement for Super-Tiny Function Calling Models' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework) or when the user references techniques from this research area."
---

# STAR: Similarity-guided Teacher-Assisted Refinement for Super-Tiny Function Calling Models

**Source:** [https://arxiv.org/abs/2602.03022v1](https://arxiv.org/abs/2602.03022v1)
**Category:** cs.AI | **Published:** 2026-02-03 | **Skill Score:** 69
**Authors:** Jiliang Ni, Jiachen Pu, Zhongyi Yang...

## Core Capability

Build and orchestrate AI agent workflows.

## Key Techniques

- **Proposed technique:** star: similarity-guided teacher-assisted refinement

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

> The proliferation of Large Language Models (LLMs) in function calling is pivotal for creating advanced AI agents, yet their large scale hinders widespread adoption, necessitating transferring their capabilities into smaller ones. However, existing paradigms are often plagued by overfitting, training instability, ineffective binary rewards for multi-solution tasks, and the difficulty of synergizing techniques. We introduce STAR: Similarity-guided Teacher-Assisted Refinement, a novel holistic fram

Refer to the [full paper](https://arxiv.org/abs/2602.03022v1) for detailed methodology.