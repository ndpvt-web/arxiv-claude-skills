---
name: "the-side-effects-of"
description: "As Multimodal Large Language Models (MLLMs) acquire stronger reasoning capabilities to handle complex, multi-image instructions, this advancement may pose new safety risks. Implements techniques from the paper 'The Side Effects of Being Smart: Safety Risks in MLLMs' Multi-Image Reasoning' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# The Side Effects of Being Smart: Safety Risks in MLLMs' Multi-Image Reasoning

**Source:** [https://arxiv.org/abs/2601.14127v1](https://arxiv.org/abs/2601.14127v1)
**Category:** cs.CV | **Published:** 2026-01-20 | **Skill Score:** 75
**Authors:** Renmiao Chen, Yida Lu, Shiyao Cui...

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

> As Multimodal Large Language Models (MLLMs) acquire stronger reasoning capabilities to handle complex, multi-image instructions, this advancement may pose new safety risks. We study this problem by introducing MIR-SafetyBench, the first benchmark focused on multi-image reasoning safety, which consists of 2,676 instances across a taxonomy of 9 multi-image relations. Our extensive evaluations on 19 MLLMs reveal a troubling trend: models with more advanced multi-image reasoning can be more vulnerab

Refer to the [full paper](https://arxiv.org/abs/2601.14127v1) for detailed methodology.