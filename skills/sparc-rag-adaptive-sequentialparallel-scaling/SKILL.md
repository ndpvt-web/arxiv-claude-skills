---
name: "sparc-rag-adaptive-sequentialparallel-scaling"
description: "Retrieval-Augmented Generation (RAG) grounds large language model outputs in external evidence, but remains challenged on multi-hop question answering that requires long reasoning. Implements techniques from the paper 'SPARC-RAG: Adaptive Sequential-Parallel Scaling with Context Management for Retrieval-Augmented Generation' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# SPARC-RAG: Adaptive Sequential-Parallel Scaling with Context Management for Retrieval-Augmented Generation

**Source:** [https://arxiv.org/abs/2602.00083v1](https://arxiv.org/abs/2602.00083v1)
**Category:** cs.IR | **Published:** 2026-01-22 | **Skill Score:** 87
**Authors:** Yuxin Yang, Gangda Deng, Ömer Faruk Akgül...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Retrieval-augmented** approach for grounding responses in external knowledge

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

> Retrieval-Augmented Generation (RAG) grounds large language model outputs in external evidence, but remains challenged on multi-hop question answering that requires long reasoning. Recent works scale RAG at inference time along two complementary dimensions: sequential depth for iterative refinement and parallel width for coverage expansion. However, naive scaling causes context contamination and scaling inefficiency, leading to diminishing or negative returns despite increased computation. To ad

Refer to the [full paper](https://arxiv.org/abs/2602.00083v1) for detailed methodology.