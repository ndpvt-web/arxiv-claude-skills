---
name: "thinking-with-geometry-active"
description: "Recent progress in spatial reasoning with Multimodal Large Language Models (MLLMs) increasingly leverages geometric priors from 3D encoders. Implements techniques from the paper 'Thinking with Geometry: Active Geometry Integration for Spatial Reasoning' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# Thinking with Geometry: Active Geometry Integration for Spatial Reasoning

**Source:** [https://arxiv.org/abs/2602.06037v2](https://arxiv.org/abs/2602.06037v2)
**Category:** cs.CV | **Published:** 2026-02-05 | **Skill Score:** 61
**Authors:** Haoyuan Li, Qihang Cao, Tao Tang...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Leverages:** geometric priors from 3d encoders

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

> Recent progress in spatial reasoning with Multimodal Large Language Models (MLLMs) increasingly leverages geometric priors from 3D encoders. However, most existing integration strategies remain passive: geometry is exposed as a global stream and fused in an indiscriminate manner, which often induces semantic-geometry misalignment and redundant signals. We propose GeoThinker, a framework that shifts the paradigm from passive fusion to active perception. Instead of feature mixing, GeoThinker enabl

Refer to the [full paper](https://arxiv.org/abs/2602.06037v2) for detailed methodology.