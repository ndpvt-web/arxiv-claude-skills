---
name: "rethinking-llmasajudge-representationasajudge-with"
description: "Large language models (LLMs) are widely used as reference-free evaluators via prompting, but this \"LLM-as-a-Judge\" paradigm is costly, opaque, and sensitive to prompt design. Implements techniques from the paper 'Rethinking LLM-as-a-Judge: Representation-as-a-Judge with Small Language Models via Semantic Capacity Asymmetry' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# Rethinking LLM-as-a-Judge: Representation-as-a-Judge with Small Language Models via Semantic Capacity Asymmetry

**Source:** [https://arxiv.org/abs/2601.22588v1](https://arxiv.org/abs/2601.22588v1)
**Category:** cs.CL | **Published:** 2026-01-30 | **Skill Score:** 80
**Authors:** Zhuochun Li, Yong Zhang, Ming Li...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Leverages:** internal representations instead of surface generation

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

> Large language models (LLMs) are widely used as reference-free evaluators via prompting, but this "LLM-as-a-Judge" paradigm is costly, opaque, and sensitive to prompt design. In this work, we investigate whether smaller models can serve as efficient evaluators by leveraging internal representations instead of surface generation. We uncover a consistent empirical pattern: small LMs, despite with weak generative ability, encode rich evaluative signals in their hidden states. This motivates us to p

Refer to the [full paper](https://arxiv.org/abs/2601.22588v1) for detailed methodology.