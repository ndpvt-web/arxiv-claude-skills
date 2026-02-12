---
name: "beyond-quantity-trajectory-diversity"
description: "As code large language models (LLMs) evolve into tool-interactive agents via the Model Context Protocol (MCP), their generalization is increasingly limited by low-quality synthetic data and the dim... Implements techniques from the paper 'Beyond Quantity: Trajectory Diversity Scaling for Code Agents' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework) or when the user references techniques from this research area."
---

# Beyond Quantity: Trajectory Diversity Scaling for Code Agents

**Source:** [https://arxiv.org/abs/2602.03219v2](https://arxiv.org/abs/2602.03219v2)
**Category:** cs.AI | **Published:** 2026-02-03 | **Skill Score:** 59
**Authors:** Guhong Chen, Chenghao Sun, Cheng Fu...

## Core Capability

Build and orchestrate AI agent workflows.

## Key Techniques

- **Leverages:** trajectory data

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

> As code large language models (LLMs) evolve into tool-interactive agents via the Model Context Protocol (MCP), their generalization is increasingly limited by low-quality synthetic data and the diminishing returns of quantity scaling. Moreover, quantity-centric scaling exhibits an early bottleneck that underutilizes trajectory data. We propose TDScaling, a Trajectory Diversity Scaling-based data synthesis framework for code agents that scales performance through diversity rather than raw volume.

Refer to the [full paper](https://arxiv.org/abs/2602.03219v2) for detailed methodology.