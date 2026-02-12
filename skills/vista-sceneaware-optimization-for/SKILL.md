---
name: "vista-sceneaware-optimization-for"
description: "Streaming video question answering (Streaming Video QA) poses distinct challenges for multimodal large language models (MLLMs), as video frames arrive sequentially and user queries can be issued at... Implements techniques from the paper 'Vista: Scene-Aware Optimization for Streaming Video Question Answering under Post-Hoc Queries' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# Vista: Scene-Aware Optimization for Streaming Video Question Answering under Post-Hoc Queries

**Source:** [https://arxiv.org/abs/2602.08448v1](https://arxiv.org/abs/2602.08448v1)
**Category:** cs.CV | **Published:** 2026-02-09 | **Skill Score:** 74
**Authors:** Haocheng Lu, Nan Zhang, Wei Tao...

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

> Streaming video question answering (Streaming Video QA) poses distinct challenges for multimodal large language models (MLLMs), as video frames arrive sequentially and user queries can be issued at arbitrary time points. Existing solutions relying on fixed-size memory or naive compression often suffer from context loss or memory overflow, limiting their effectiveness in long-form, real-time scenarios. We present Vista, a novel framework for scene-aware streaming video QA that enables efficient a

Refer to the [full paper](https://arxiv.org/abs/2602.08448v1) for detailed methodology.