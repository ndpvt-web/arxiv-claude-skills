---
name: "knowledge-graphs-are-implicit"
description: "Large language models have achieved near-expert performance in structured reasoning domains like mathematics and programming, yet their ability to perform compositional multi-hop reasoning in speci... Implements techniques from the paper 'Knowledge Graphs are Implicit Reward Models: Path-Derived Signals Enable Compositional Reasoning' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# Knowledge Graphs are Implicit Reward Models: Path-Derived Signals Enable Compositional Reasoning

**Source:** [https://arxiv.org/abs/2601.15160v1](https://arxiv.org/abs/2601.15160v1)
**Category:** cs.AI | **Published:** 2026-01-21 | **Skill Score:** 72
**Authors:** Yuval Kansal, Niraj K. Jha

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** a bottom-up learning paradigm in which models are grounded in axiomatic domain facts and compose them to solve complex
- **Proposed technique:** a post-training pipeline

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

> Large language models have achieved near-expert performance in structured reasoning domains like mathematics and programming, yet their ability to perform compositional multi-hop reasoning in specialized scientific fields remains limited. We propose a bottom-up learning paradigm in which models are grounded in axiomatic domain facts and compose them to solve complex, unseen tasks. To this end, we present a post-training pipeline, based on a combination of supervised fine-tuning and reinforcement

Refer to the [full paper](https://arxiv.org/abs/2601.15160v1) for detailed methodology.