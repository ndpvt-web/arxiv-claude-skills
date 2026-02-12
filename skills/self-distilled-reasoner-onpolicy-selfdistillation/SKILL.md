---
name: "self-distilled-reasoner-onpolicy-selfdistillation"
description: "Knowledge distillation improves large language model (LLM) reasoning by compressing the knowledge of a teacher LLM to train smaller LLMs. Implements techniques from the paper 'Self-Distilled Reasoner: On-Policy Self-Distillation for Large Language Models' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# Self-Distilled Reasoner: On-Policy Self-Distillation for Large Language Models

**Source:** [https://arxiv.org/abs/2601.18734v1](https://arxiv.org/abs/2601.18734v1)
**Category:** cs.LG | **Published:** 2026-01-26 | **Skill Score:** 66
**Authors:** Siyan Zhao, Zhihui Xie, Mengchen Liu...

## Core Capability

Search, retrieve, and synthesize information.

## Workflow

1. Parse the user's information need or query
2. Formulate effective search strategies
3. Retrieve relevant documents, code, or data
4. Synthesize findings into a coherent response
5. Provide citations and references

## Agent Coordination

- Decompose complex tasks into independent subtasks where possible
- Use parallel execution for independent subtasks
- Implement error recovery and retry logic
- Aggregate partial results when full completion fails

## Research Context

> Knowledge distillation improves large language model (LLM) reasoning by compressing the knowledge of a teacher LLM to train smaller LLMs. On-policy distillation advances this approach by having the student sample its own trajectories while a teacher LLM provides dense token-level supervision, addressing the distribution mismatch between training and inference in off-policy distillation methods. However, on-policy distillation typically requires a separate, often larger, teacher LLM and does not 

Refer to the [full paper](https://arxiv.org/abs/2601.18734v1) for detailed methodology.