---
name: "tracemem-weaving-narrative-memory"
description: "Sustaining long-term interactions remains a bottleneck for Large Language Models (LLMs), as their limited context windows struggle to manage dialogue histories that extend over time. Implements techniques from the paper 'TraceMem: Weaving Narrative Memory Schemata from User Conversational Traces' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework), (database & query) or when the user references techniques from this research area."
---

# TraceMem: Weaving Narrative Memory Schemata from User Conversational Traces

**Source:** [https://arxiv.org/abs/2602.09712v1](https://arxiv.org/abs/2602.09712v1)
**Category:** cs.CL | **Published:** 2026-02-10 | **Skill Score:** 95
**Authors:** Yiming Shu, Pei Liu, Tiange Zhang...

## Core Capability

Search, retrieve, and synthesize information.

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

> Sustaining long-term interactions remains a bottleneck for Large Language Models (LLMs), as their limited context windows struggle to manage dialogue histories that extend over time. Existing memory systems often treat interactions as disjointed snippets, failing to capture the underlying narrative coherence of the dialogue stream. We propose TraceMem, a cognitively-inspired framework that weaves structured, narrative memory schemata from user conversational traces through a three-stage pipeline

Refer to the [full paper](https://arxiv.org/abs/2602.09712v1) for detailed methodology.