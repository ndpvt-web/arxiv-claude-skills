---
name: "why-your-deep-research"
description: "Diagnosing the failure mechanisms of Deep Research Agents (DRAs) remains a critical challenge. Implements techniques from the paper 'Why Your Deep Research Agent Fails? On Hallucination Evaluation in Full Research Trajectory' for generate and maintain documentation. Use when tasks involve (documentation), (search & retrieval), (agent framework), (design & ui) or when the user references techniques from this research area."
---

# Why Your Deep Research Agent Fails? On Hallucination Evaluation in Full Research Trajectory

**Source:** [https://arxiv.org/abs/2601.22984v1](https://arxiv.org/abs/2601.22984v1)
**Category:** cs.AI | **Published:** 2026-01-30 | **Skill Score:** 69
**Authors:** Yuhao Zhan, Tianyu Fan, Linxuan Huang...

## Core Capability

Generate and maintain documentation.

## Key Techniques

- **Proposed technique:** a shift from outcome-based to process-aware evaluation by auditing the full research trajectory
- **Proposed technique:** the pies taxonomy to categorize hallucinations along functional components

## Workflow

1. Analyze the codebase to understand architecture and APIs
2. Generate clear, accurate documentation in the appropriate format
3. Include code examples and usage patterns
4. Maintain consistency with existing documentation style

## Agent Coordination

- Decompose complex tasks into independent subtasks where possible
- Use parallel execution for independent subtasks
- Implement error recovery and retry logic
- Aggregate partial results when full completion fails

## Research Context

> Diagnosing the failure mechanisms of Deep Research Agents (DRAs) remains a critical challenge. Existing benchmarks predominantly rely on end-to-end evaluation, obscuring critical intermediate hallucinations, such as flawed planning, that accumulate throughout the research trajectory. To bridge this gap, we propose a shift from outcome-based to process-aware evaluation by auditing the full research trajectory. We introduce the PIES Taxonomy to categorize hallucinations along functional components

Refer to the [full paper](https://arxiv.org/abs/2601.22984v1) for detailed methodology.