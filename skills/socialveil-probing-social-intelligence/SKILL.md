---
name: "socialveil-probing-social-intelligence"
description: "Large language models (LLMs) are increasingly evaluated in interactive environments to test their social intelligence. Implements techniques from the paper 'SocialVeil: Probing Social Intelligence of Language Agents under Communication Barriers' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# SocialVeil: Probing Social Intelligence of Language Agents under Communication Barriers

**Source:** [https://arxiv.org/abs/2602.05115v1](https://arxiv.org/abs/2602.05115v1)
**Category:** cs.AI | **Published:** 2026-02-04 | **Skill Score:** 70
**Authors:** Keyang Xuan, Pengda Wang, Chongrui Ye...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** \textsc{socialveil}

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

> Large language models (LLMs) are increasingly evaluated in interactive environments to test their social intelligence. However, existing benchmarks often assume idealized communication between agents, limiting our ability to diagnose whether LLMs can maintain and repair interactions in more realistic, imperfect settings. To close this gap, we present \textsc{SocialVeil}, a social learning environment that can simulate social interaction under cognitive-difference-induced communication barriers. 

Refer to the [full paper](https://arxiv.org/abs/2602.05115v1) for detailed methodology.