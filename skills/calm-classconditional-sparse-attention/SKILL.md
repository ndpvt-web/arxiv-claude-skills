---
name: "calm-classconditional-sparse-attention"
description: "Large audio-language models (LALMs) exhibit strong zero-shot capabilities in multiple downstream tasks, such as audio question answering (AQA) and abstract reasoning; however, these models still la... Implements techniques from the paper 'CALM: Class-Conditional Sparse Attention Vectors for Large Audio-Language Models' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (content generation), (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# CALM: Class-Conditional Sparse Attention Vectors for Large Audio-Language Models

**Source:** [https://arxiv.org/abs/2602.07077v1](https://arxiv.org/abs/2602.07077v1)
**Category:** cs.SD | **Published:** 2026-02-06 | **Skill Score:** 64
**Authors:** Videet Mehta, Liming Wang, Hilde Kuehne...

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

> Large audio-language models (LALMs) exhibit strong zero-shot capabilities in multiple downstream tasks, such as audio question answering (AQA) and abstract reasoning; however, these models still lag behind specialized models for certain discriminative tasks (e.g., audio classification). Recent studies show that sparse subsets of attention heads within an LALM can serve as strong discriminative feature extractors for downstream tasks such as classification via simple voting schemes. However, thes

Refer to the [full paper](https://arxiv.org/abs/2602.07077v1) for detailed methodology.