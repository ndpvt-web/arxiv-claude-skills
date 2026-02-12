---
name: "steuerllm-local-specialized-large"
description: "Large language models (LLMs) demonstrate strong general reasoning and language understanding, yet their performance degrades in domains governed by strict formal rules, precise terminology, and leg... Implements techniques from the paper 'SteuerLLM: Local specialized large language model for German tax law analysis' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# SteuerLLM: Local specialized large language model for German tax law analysis

**Source:** [https://arxiv.org/abs/2602.11081v1](https://arxiv.org/abs/2602.11081v1)
**Category:** cs.CL | **Published:** 2026-02-11 | **Skill Score:** 95
**Authors:** Sebastian Wind, Jeta Sopa, Laurin Schmid...

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

> Large language models (LLMs) demonstrate strong general reasoning and language understanding, yet their performance degrades in domains governed by strict formal rules, precise terminology, and legally binding structure. Tax law exemplifies these challenges, as correct answers require exact statutory citation, structured legal argumentation, and numerical accuracy under rigid grading schemes. We algorithmically generate SteuerEx, the first open benchmark derived from authentic German university 

Refer to the [full paper](https://arxiv.org/abs/2602.11081v1) for detailed methodology.