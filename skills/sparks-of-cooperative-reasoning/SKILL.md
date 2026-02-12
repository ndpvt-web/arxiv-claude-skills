---
name: "sparks-of-cooperative-reasoning"
description: "Cooperative reasoning under incomplete information remains challenging for both humans and multi-agent systems. Implements techniques from the paper 'Sparks of Cooperative Reasoning: LLMs as Strategic Hanabi Agents' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# Sparks of Cooperative Reasoning: LLMs as Strategic Hanabi Agents

**Source:** [https://arxiv.org/abs/2601.18077v1](https://arxiv.org/abs/2601.18077v1)
**Category:** cs.CL | **Published:** 2026-01-26 | **Skill Score:** 63
**Authors:** Mahesh Ramesh, Kaousheik Jayakumar, Aswinkumar Ramkumar...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Multi-agent architecture** for task decomposition and parallel execution

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

> Cooperative reasoning under incomplete information remains challenging for both humans and multi-agent systems. The card game Hanabi embodies this challenge, requiring theory-of-mind reasoning and strategic communication. We benchmark 17 state-of-the-art LLM agents in 2-5 player games and study the impact of context engineering across model scales (4B to 600B+) to understand persistent coordination failures and robustness to scaffolding: from a minimal prompt with only explicit card details (Wat

Refer to the [full paper](https://arxiv.org/abs/2601.18077v1) for detailed methodology.