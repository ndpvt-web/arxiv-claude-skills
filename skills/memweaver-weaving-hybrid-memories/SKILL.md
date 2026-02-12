---
name: "memweaver-weaving-hybrid-memories"
description: "Large language model-based agents operating in long-horizon interactions require memory systems that support temporal consistency, multi-hop reasoning, and evidence-grounded reuse across sessions. Implements techniques from the paper 'MemWeaver: Weaving Hybrid Memories for Traceable Long-Horizon Agentic Reasoning' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework), (design & ui) or when the user references techniques from this research area."
---

# MemWeaver: Weaving Hybrid Memories for Traceable Long-Horizon Agentic Reasoning

**Source:** [https://arxiv.org/abs/2601.18204v1](https://arxiv.org/abs/2601.18204v1)
**Category:** cs.CL | **Published:** 2026-01-26 | **Skill Score:** 60
**Authors:** Juexiang Ye, Xue Li, Xinyu Yang...

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

> Large language model-based agents operating in long-horizon interactions require memory systems that support temporal consistency, multi-hop reasoning, and evidence-grounded reuse across sessions. Existing approaches largely rely on unstructured retrieval or coarse abstractions, which often lead to temporal conflicts, brittle reasoning, and limited traceability. We propose MemWeaver, a unified memory framework that consolidates long-term agent experiences into three interconnected components: a 

Refer to the [full paper](https://arxiv.org/abs/2601.18204v1) for detailed methodology.