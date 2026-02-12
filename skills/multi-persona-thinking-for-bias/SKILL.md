---
name: "multi-persona-thinking-for-bias"
description: "Large Language Models (LLMs) exhibit significant social biases that can perpetuate harmful stereotypes and unfair outcomes. Implements techniques from the paper 'Multi-Persona Thinking for Bias Mitigation in Large Language Models' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# Multi-Persona Thinking for Bias Mitigation in Large Language Models

**Source:** [https://arxiv.org/abs/2601.15488v1](https://arxiv.org/abs/2601.15488v1)
**Category:** cs.CL | **Published:** 2026-01-21 | **Skill Score:** 65
**Authors:** Yuxing Chen, Guoqing Luo, Zijun Wu...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** multi-persona thinking (mpt)
- **Novel approach:** inference-time framework
- **Leverages:** dialectical reasoning from multiple perspectives to reduce bias

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

> Large Language Models (LLMs) exhibit significant social biases that can perpetuate harmful stereotypes and unfair outcomes. In this paper, we propose Multi-Persona Thinking (MPT), a novel inference-time framework that leverages dialectical reasoning from multiple perspectives to reduce bias. MPT guides models to adopt contrasting social identities (e.g., male and female) along with a neutral viewpoint, and then engages these personas iteratively to expose and correct biases. Through a dialectica

Refer to the [full paper](https://arxiv.org/abs/2601.15488v1) for detailed methodology.