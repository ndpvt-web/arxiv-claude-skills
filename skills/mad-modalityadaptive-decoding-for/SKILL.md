---
name: "mad-modalityadaptive-decoding-for"
description: "Multimodal Large Language Models (MLLMs) suffer from cross-modal hallucinations, where one modality inappropriately influences generation about another, leading to fabricated output. Implements techniques from the paper 'MAD: Modality-Adaptive Decoding for Mitigating Cross-Modal Hallucinations in Multimodal Large Language Models' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (content generation), (agent framework) or when the user references techniques from this research area."
---

# MAD: Modality-Adaptive Decoding for Mitigating Cross-Modal Hallucinations in Multimodal Large Language Models

**Source:** [https://arxiv.org/abs/2601.21181v1](https://arxiv.org/abs/2601.21181v1)
**Category:** cs.AI | **Published:** 2026-01-29 | **Skill Score:** 68
**Authors:** Sangyun Chung, Se Yeon Kim, Youngchae Chee...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** modality-adaptive decoding (mad)
- **Leverages:** the model's inherent ability to self-assess modality r

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

> Multimodal Large Language Models (MLLMs) suffer from cross-modal hallucinations, where one modality inappropriately influences generation about another, leading to fabricated output. This exposes a more fundamental deficiency in modality-interaction control. To address this, we propose Modality-Adaptive Decoding (MAD), a training-free method that adaptively weights modality-specific decoding branches based on task requirements. MAD leverages the model's inherent ability to self-assess modality r

Refer to the [full paper](https://arxiv.org/abs/2601.21181v1) for detailed methodology.