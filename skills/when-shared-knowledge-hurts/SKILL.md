---
name: "when-shared-knowledge-hurts"
description: "Model merging combines multiple fine-tuned models into a single model by adding their weight updates, providing a lightweight alternative to retraining. Implements techniques from the paper 'When Shared Knowledge Hurts: Spectral Over-Accumulation in Model Merging' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval) or when the user references techniques from this research area."
---

# When Shared Knowledge Hurts: Spectral Over-Accumulation in Model Merging

**Source:** [https://arxiv.org/abs/2602.05536v1](https://arxiv.org/abs/2602.05536v1)
**Category:** cs.LG | **Published:** 2026-02-05 | **Skill Score:** 60
**Authors:** Yayuan Li, Ze Peng, Jian Zhang...

## Core Capability

Search, retrieve, and synthesize information.

## Workflow

1. Parse the user's information need or query
2. Formulate effective search strategies
3. Retrieve relevant documents, code, or data
4. Synthesize findings into a coherent response
5. Provide citations and references

## Research Context

> Model merging combines multiple fine-tuned models into a single model by adding their weight updates, providing a lightweight alternative to retraining. Existing methods primarily target resolving conflicts between task updates, leaving the failure mode of over-counting shared knowledge unaddressed. We show that when tasks share aligned spectral directions (i.e., overlapping singular vectors), a simple linear combination repeatedly accumulates these directions, inflating the singular values and 

Refer to the [full paper](https://arxiv.org/abs/2602.05536v1) for detailed methodology.