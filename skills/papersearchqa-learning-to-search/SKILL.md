---
name: "papersearchqa-learning-to-search"
description: "Search agents are language models (LMs) that reason and search knowledge bases (or the web) to answer questions; recent methods supervise only the final answer accuracy using reinforcement learning... Implements techniques from the paper 'PaperSearchQA: Learning to Search and Reason over Scientific Papers with RLVR' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# PaperSearchQA: Learning to Search and Reason over Scientific Papers with RLVR

**Source:** [https://arxiv.org/abs/2601.18207v1](https://arxiv.org/abs/2601.18207v1)
**Category:** cs.LG | **Published:** 2026-01-26 | **Skill Score:** 75
**Authors:** James Burgess, Jan N. Hansen, Duo Peng...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** training agents to search and reason over scientific papers -- this tests technical question-answering

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

> Search agents are language models (LMs) that reason and search knowledge bases (or the web) to answer questions; recent methods supervise only the final answer accuracy using reinforcement learning with verifiable rewards (RLVR). Most RLVR search agents tackle general-domain QA, which limits their relevance to technical AI systems in science, engineering, and medicine. In this work we propose training agents to search and reason over scientific papers -- this tests technical question-answering, 

Refer to the [full paper](https://arxiv.org/abs/2601.18207v1) for detailed methodology.