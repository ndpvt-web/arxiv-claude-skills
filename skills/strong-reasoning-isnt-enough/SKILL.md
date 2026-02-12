---
name: "strong-reasoning-isnt-enough"
description: "Interactive medical consultation requires an agent to proactively elicit missing clinical evidence under uncertainty. Implements techniques from the paper 'Strong Reasoning Isn't Enough: Evaluating Evidence Elicitation in Interactive Diagnosis' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# Strong Reasoning Isn't Enough: Evaluating Evidence Elicitation in Interactive Diagnosis

**Source:** [https://arxiv.org/abs/2601.19773v1](https://arxiv.org/abs/2601.19773v1)
**Category:** cs.CL | **Published:** 2026-01-27 | **Skill Score:** 67
**Authors:** Zhuohan Long, Zhijie Bao, Zhongyu Wei

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** information coverage rate (icr) t

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

> Interactive medical consultation requires an agent to proactively elicit missing clinical evidence under uncertainty. Yet existing evaluations largely remain static or outcome-centric, neglecting the evidence-gathering process. In this work, we propose an interactive evaluation framework that explicitly models the consultation process using a simulated patient and a \rev{simulated reporter} grounded in atomic evidences. Based on this representation, we introduce Information Coverage Rate (ICR) t

Refer to the [full paper](https://arxiv.org/abs/2601.19773v1) for detailed methodology.