---
name: "sdr-cir-semantic-debias-retrieval"
description: "Composed Image Retrieval (CIR) aims to retrieve a target image from a query composed of a reference image and modification text. Implements techniques from the paper 'SDR-CIR: Semantic Debias Retrieval Framework for Training-Free Zero-Shot Composed Image Retrieval' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# SDR-CIR: Semantic Debias Retrieval Framework for Training-Free Zero-Shot Composed Image Retrieval

**Source:** [https://arxiv.org/abs/2602.04451v2](https://arxiv.org/abs/2602.04451v2)
**Category:** cs.IR | **Published:** 2026-02-04 | **Skill Score:** 75
**Authors:** Yi Sun, Jinyu Xu, Qing Xie...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Retrieval-augmented** approach for grounding responses in external knowledge
- **Chain-of-thought reasoning** for improved step-by-step problem solving

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

> Composed Image Retrieval (CIR) aims to retrieve a target image from a query composed of a reference image and modification text. Recent training-free zero-shot methods often employ Multimodal Large Language Models (MLLMs) with Chain-of-Thought (CoT) to compose a target image description for retrieval. However, due to the fuzzy matching nature of ZS-CIR, the generated description is prone to semantic bias relative to the target image. We propose SDR-CIR, a training-free Semantic Debias Ranking me

Refer to the [full paper](https://arxiv.org/abs/2602.04451v2) for detailed methodology.