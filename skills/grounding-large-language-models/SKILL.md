---
name: "grounding-large-language-models"
description: "Large Language Models (LLMs) can aid synthesis planning in chemistry, but standard prompting methods often yield hallucinated or outdated suggestions. Implements techniques from the paper 'Grounding Large Language Models in Reaction Knowledge Graphs for Synthesis Retrieval' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# Grounding Large Language Models in Reaction Knowledge Graphs for Synthesis Retrieval

**Source:** [https://arxiv.org/abs/2601.16038v1](https://arxiv.org/abs/2601.16038v1)
**Category:** cs.AI | **Published:** 2026-01-22 | **Skill Score:** 74
**Authors:** Olga Bunkova, Lorenzo Di Fruscia, Sophia Rupprecht...

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

> Large Language Models (LLMs) can aid synthesis planning in chemistry, but standard prompting methods often yield hallucinated or outdated suggestions. We study LLM interactions with a reaction knowledge graph by casting reaction path retrieval as a Text2Cypher (natural language to graph query) generation problem, and define single- and multi-step retrieval tasks. We compare zero-shot prompting to one-shot variants using static, random, and embedding-based exemplar selection, and assess a checkli

Refer to the [full paper](https://arxiv.org/abs/2601.16038v1) for detailed methodology.