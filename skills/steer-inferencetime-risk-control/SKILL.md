---
name: "steer-inferencetime-risk-control"
description: "Large Language Models (LLMs) trained for average correctness often exhibit mode collapse, producing narrow decision behaviors on tasks where multiple responses may be reasonable. Implements techniques from the paper 'STEER: Inference-Time Risk Control via Constrained Quality-Diversity Search' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# STEER: Inference-Time Risk Control via Constrained Quality-Diversity Search

**Source:** [https://arxiv.org/abs/2602.02862v1](https://arxiv.org/abs/2602.02862v1)
**Category:** cs.AI | **Published:** 2026-02-02 | **Skill Score:** 59
**Authors:** Eric Yang, Jong Ha Lee, Jonathan Amar...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** steer (steerable tuning via evolutionary ensemble refinement)

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

> Large Language Models (LLMs) trained for average correctness often exhibit mode collapse, producing narrow decision behaviors on tasks where multiple responses may be reasonable. This limitation is particularly problematic in ordinal decision settings such as clinical triage, where standard alignment removes the ability to trade off specificity and sensitivity (the ROC operating point) based on contextual constraints. We propose STEER (Steerable Tuning via Evolutionary Ensemble Refinement), a tr

Refer to the [full paper](https://arxiv.org/abs/2602.02862v1) for detailed methodology.