---
name: "toward-culturally-aligned-llms"
description: "Large Language Models (LLMs) increasingly support culturally sensitive decision making, yet often exhibit misalignment due to skewed pretraining data and the absence of structured value representat... Implements techniques from the paper 'Toward Culturally Aligned LLMs through Ontology-Guided Multi-Agent Reasoning' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework) or when the user references techniques from this research area."
---

# Toward Culturally Aligned LLMs through Ontology-Guided Multi-Agent Reasoning

**Source:** [https://arxiv.org/abs/2601.21700v2](https://arxiv.org/abs/2601.21700v2)
**Category:** cs.CL | **Published:** 2026-01-29 | **Skill Score:** 90
**Authors:** Wonduk Seo, Wonseok Choi, Junseo Koh...

## Core Capability

Build and orchestrate AI agent workflows.

## Key Techniques

- **Multi-agent architecture** for task decomposition and parallel execution

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

> Large Language Models (LLMs) increasingly support culturally sensitive decision making, yet often exhibit misalignment due to skewed pretraining data and the absence of structured value representations. Existing methods can steer outputs, but often lack demographic grounding and treat values as independent, unstructured signals, reducing consistency and interpretability. We propose OG-MAR, an Ontology-Guided Multi-Agent Reasoning framework. OG-MAR summarizes respondent-specific values from the W

Refer to the [full paper](https://arxiv.org/abs/2601.21700v2) for detailed methodology.