---
name: "ovd-onpolicy-verbal-distillation"
description: "Knowledge distillation offers a promising path to transfer reasoning capabilities from large teacher models to efficient student models; however, existing token-level on-policy distillation methods... Implements techniques from the paper 'OVD: On-policy Verbal Distillation' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# OVD: On-policy Verbal Distillation

**Source:** [https://arxiv.org/abs/2601.21968v1](https://arxiv.org/abs/2601.21968v1)
**Category:** cs.CL | **Published:** 2026-01-29 | **Skill Score:** 63
**Authors:** Jing Xiong, Hui Shen, Shansan Gong...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** on-policy verbal distillation (ovd

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

> Knowledge distillation offers a promising path to transfer reasoning capabilities from large teacher models to efficient student models; however, existing token-level on-policy distillation methods require token-level alignment between the student and teacher models, which restricts the student model's exploration ability, prevent effective use of interactive environment feedback, and suffer from severe memory bottlenecks in reinforcement learning. We introduce On-policy Verbal Distillation (OVD

Refer to the [full paper](https://arxiv.org/abs/2601.21968v1) for detailed methodology.