---
name: "aero-autonomous-evolutionary-reasoning"
description: "Large Language Models (LLMs) have achieved significant success in complex reasoning but remain bottlenecked by reliance on expert-annotated data and external verifiers. Implements techniques from the paper 'AERO: Autonomous Evolutionary Reasoning Optimization via Endogenous Dual-Loop Feedback' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# AERO: Autonomous Evolutionary Reasoning Optimization via Endogenous Dual-Loop Feedback

**Source:** [https://arxiv.org/abs/2602.03084v2](https://arxiv.org/abs/2602.03084v2)
**Category:** cs.CL | **Published:** 2026-02-03 | **Skill Score:** 74
**Authors:** Zhitao Gao, Jie Ma, Xuhong Li...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** \underline{a}utonomous \underline{e}volutionary \underline{r}e

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

> Large Language Models (LLMs) have achieved significant success in complex reasoning but remain bottlenecked by reliance on expert-annotated data and external verifiers. While existing self-evolution paradigms aim to bypass these constraints, they often fail to identify the optimal learning zone and risk reinforcing collective hallucinations and incorrect priors through flawed internal feedback. To address these challenges, we propose \underline{A}utonomous \underline{E}volutionary \underline{R}e

Refer to the [full paper](https://arxiv.org/abs/2602.03084v2) for detailed methodology.