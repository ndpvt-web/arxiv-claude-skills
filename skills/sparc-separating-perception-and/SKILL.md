---
name: "sparc-separating-perception-and"
description: "Despite recent successes, test-time scaling - i.e., dynamically expanding the token budget during inference as needed - remains brittle for vision-language models (VLMs): unstructured chains-of-tho... Implements techniques from the paper 'SPARC: Separating Perception And Reasoning Circuits for Test-time Scaling of VLMs' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# SPARC: Separating Perception And Reasoning Circuits for Test-time Scaling of VLMs

**Source:** [https://arxiv.org/abs/2602.06566v2](https://arxiv.org/abs/2602.06566v2)
**Category:** cs.CV | **Published:** 2026-02-06 | **Skill Score:** 71
**Authors:** Niccolo Avogaro, Nayanika Debnath, Li Mi...

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

> Despite recent successes, test-time scaling - i.e., dynamically expanding the token budget during inference as needed - remains brittle for vision-language models (VLMs): unstructured chains-of-thought about images entangle perception and reasoning, leading to long, disorganized contexts where small perceptual mistakes may cascade into completely wrong answers. Moreover, expensive reinforcement learning with hand-crafted rewards is required to achieve good performance. Here, we introduce SPARC (

Refer to the [full paper](https://arxiv.org/abs/2602.06566v2) for detailed methodology.