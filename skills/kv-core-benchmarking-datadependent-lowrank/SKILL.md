---
name: "kv-core-benchmarking-datadependent-lowrank"
description: "Large language models rely on kv-caches to avoid redundant computation during autoregressive decoding, but as context length grows, reading and writing the cache can quickly saturate GPU memory ban... Implements techniques from the paper 'KV-CoRE: Benchmarking Data-Dependent Low-Rank Compressibility of KV-Caches in LLMs' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval) or when the user references techniques from this research area."
---

# KV-CoRE: Benchmarking Data-Dependent Low-Rank Compressibility of KV-Caches in LLMs

**Source:** [https://arxiv.org/abs/2602.05929v2](https://arxiv.org/abs/2602.05929v2)
**Category:** cs.CL | **Published:** 2026-02-05 | **Skill Score:** 68
**Authors:** Jian Chen, Zhuoran Wang, Jiayu Qin...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** kv-core kv-cache compressibility by rank evaluation)

## Workflow

1. Parse the user's information need or query
2. Formulate effective search strategies
3. Retrieve relevant documents, code, or data
4. Synthesize findings into a coherent response
5. Provide citations and references

## Research Context

> Large language models rely on kv-caches to avoid redundant computation during autoregressive decoding, but as context length grows, reading and writing the cache can quickly saturate GPU memory bandwidth. Recent work has explored KV-cache compression, yet most approaches neglect the data-dependent nature of kv-caches and their variation across layers. We introduce KV-CoRE KV-cache Compressibility by Rank Evaluation), an SVD-based method for quantifying the data-dependent low-rank compressibility

Refer to the [full paper](https://arxiv.org/abs/2602.05929v2) for detailed methodology.