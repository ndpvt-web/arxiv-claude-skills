---
name: "vica-efficient-multimodal-llms"
description: "Modern multimodal large language models (MLLMs) adopt a unified self-attention design that processes visual and textual tokens at every Transformer layer, incurring substantial computational overhead. Implements techniques from the paper 'ViCA: Efficient Multimodal LLMs with Vision-Only Cross-Attention' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval) or when the user references techniques from this research area."
---

# ViCA: Efficient Multimodal LLMs with Vision-Only Cross-Attention

**Source:** [https://arxiv.org/abs/2602.07574v1](https://arxiv.org/abs/2602.07574v1)
**Category:** cs.CV | **Published:** 2026-02-07 | **Skill Score:** 82
**Authors:** Wenjie Liu, Hao Wu, Xin Qiu...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** vica (vision-on

## Workflow

1. Parse the user's information need or query
2. Formulate effective search strategies
3. Retrieve relevant documents, code, or data
4. Synthesize findings into a coherent response
5. Provide citations and references

## Research Context

> Modern multimodal large language models (MLLMs) adopt a unified self-attention design that processes visual and textual tokens at every Transformer layer, incurring substantial computational overhead. In this work, we revisit the necessity of such dense visual processing and show that projected visual embeddings are already well-aligned with the language space, while effective vision-language interaction occurs in only a small subset of layers. Based on these insights, we propose ViCA (Vision-on

Refer to the [full paper](https://arxiv.org/abs/2602.07574v1) for detailed methodology.