---
name: "workflow-r1-group-subsequence-policy"
description: "The rapid evolution of agentic workflows has demonstrated strong performance of LLM-based agents in addressing complex reasoning tasks. Implements techniques from the paper 'Workflow-R1: Group Sub-sequence Policy Optimization for Multi-turn Workflow Construction' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework) or when the user references techniques from this research area."
---

# Workflow-R1: Group Sub-sequence Policy Optimization for Multi-turn Workflow Construction

**Source:** [https://arxiv.org/abs/2602.01202v1](https://arxiv.org/abs/2602.01202v1)
**Category:** cs.AI | **Published:** 2026-02-01 | **Skill Score:** 60
**Authors:** Mingze Kong, Zikun Qu, Zhongquan Zhou...

## Core Capability

Build and orchestrate AI agent workflows.

## Key Techniques

- **Proposed technique:** workflow-r1

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

> The rapid evolution of agentic workflows has demonstrated strong performance of LLM-based agents in addressing complex reasoning tasks. However, existing workflow optimization methods typically formulate workflow synthesis as a static, one-shot code-centric generation problem. This paradigm imposes excessive constraints on the model's coding capabilities and restricts the flexibility required for dynamic problem-solving. In this paper, we present Workflow-R1, a framework that reformulates workfl

Refer to the [full paper](https://arxiv.org/abs/2602.01202v1) for detailed methodology.