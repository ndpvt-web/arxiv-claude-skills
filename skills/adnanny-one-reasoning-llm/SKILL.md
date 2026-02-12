---
name: "adnanny-one-reasoning-llm"
description: "Large Language Models (LLMs) have shown strong capabilities in Natural Language Understanding and Generation, but deploying them directly in online advertising systems is often impractical due to s... Implements techniques from the paper 'AdNanny: One Reasoning LLM for All Offline Ads Recommendation Tasks' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework), (database & query) or when the user references techniques from this research area."
---

# AdNanny: One Reasoning LLM for All Offline Ads Recommendation Tasks

**Source:** [https://arxiv.org/abs/2602.01563v1](https://arxiv.org/abs/2602.01563v1)
**Category:** cs.SE | **Published:** 2026-02-02 | **Skill Score:** 58
**Authors:** Nan Hu, Han Li, Jimeng Sun...

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

> Large Language Models (LLMs) have shown strong capabilities in Natural Language Understanding and Generation, but deploying them directly in online advertising systems is often impractical due to strict millisecond-level latency constraints. This has motivated the use of LLMs offline to improve retrieval, ranking, and recommendation models. Existing solutions typically fine-tune separate LLMs for individual tasks such as query-ad relevance labeling, keyword-based query generation, and user profi

Refer to the [full paper](https://arxiv.org/abs/2602.01563v1) for detailed methodology.