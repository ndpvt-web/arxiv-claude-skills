---
name: "focus-dllm-accelerating-longcontext-diffusion"
description: "Diffusion Large Language Models (dLLMs) deliver strong long-context processing capability in a non-autoregressive decoding paradigm. Implements techniques from the paper 'Focus-dLLM: Accelerating Long-Context Diffusion LLM Inference via Confidence-Guided Context Focusing' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval) or when the user references techniques from this research area."
---

# Focus-dLLM: Accelerating Long-Context Diffusion LLM Inference via Confidence-Guided Context Focusing

**Source:** [https://arxiv.org/abs/2602.02159v1](https://arxiv.org/abs/2602.02159v1)
**Category:** cs.CL | **Published:** 2026-02-02 | **Skill Score:** 63
**Authors:** Lingkun Long, Yushi Huang, Shihao Bai...

## Core Capability

Search, retrieve, and synthesize information.

## Workflow

1. Parse the user's information need or query
2. Formulate effective search strategies
3. Retrieve relevant documents, code, or data
4. Synthesize findings into a coherent response
5. Provide citations and references

## Research Context

> Diffusion Large Language Models (dLLMs) deliver strong long-context processing capability in a non-autoregressive decoding paradigm. However, the considerable computational cost of bidirectional full attention limits the inference efficiency. Although sparse attention is promising, existing methods remain ineffective. This stems from the need to estimate attention importance for tokens yet to be decoded, while the unmasked token positions are unknown during diffusion. In this paper, we present F

Refer to the [full paper](https://arxiv.org/abs/2602.02159v1) for detailed methodology.