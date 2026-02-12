---
name: "when-better-prompts-hurt"
description: "Evaluating Large Language Model (LLM) applications differs from traditional software testing because outputs are stochastic, high-dimensional, and sensitive to prompt and model changes. Implements techniques from the paper 'When \"Better\" Prompts Hurt: Evaluation-Driven Iteration for LLM Applications' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework), (prompt engineering), (design & ui) or when the user references techniques from this research area."
---

# When "Better" Prompts Hurt: Evaluation-Driven Iteration for LLM Applications

**Source:** [https://arxiv.org/abs/2601.22025v1](https://arxiv.org/abs/2601.22025v1)
**Category:** cs.CL | **Published:** 2026-01-29 | **Skill Score:** 100
**Authors:** Daniel Commey

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** an evaluation-driven workflow - define
- **Proposed technique:** the minimum viable evaluation suite (mves)
- **Retrieval-augmented** approach for grounding responses in external knowledge

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

> Evaluating Large Language Model (LLM) applications differs from traditional software testing because outputs are stochastic, high-dimensional, and sensitive to prompt and model changes. We present an evaluation-driven workflow - Define, Test, Diagnose, Fix - that turns these challenges into a repeatable engineering loop.   We introduce the Minimum Viable Evaluation Suite (MVES), a tiered set of recommended evaluation components for (i) general LLM applications, (ii) retrieval-augmented generatio

Refer to the [full paper](https://arxiv.org/abs/2601.22025v1) for detailed methodology.