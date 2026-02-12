---
name: "table-as-search-formulate-longhorizon-agentic"
description: "Current Information Seeking (InfoSeeking) agents struggle to maintain focus and coherence during long-horizon exploration, as tracking search states, including planning procedure and massive search... Implements techniques from the paper 'Table-as-Search: Formulate Long-Horizon Agentic Information Seeking as Table Completion' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework), (database & query) or when the user references techniques from this research area."
---

# Table-as-Search: Formulate Long-Horizon Agentic Information Seeking as Table Completion

**Source:** [https://arxiv.org/abs/2602.06724v1](https://arxiv.org/abs/2602.06724v1)
**Category:** cs.CL | **Published:** 2026-02-06 | **Skill Score:** 66
**Authors:** Tian Lan, Felix Henry, Bin Zhu...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** \textbf{table-as-search (tas)}

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

> Current Information Seeking (InfoSeeking) agents struggle to maintain focus and coherence during long-horizon exploration, as tracking search states, including planning procedure and massive search results, within one plain-text context is inherently fragile. To address this, we introduce \textbf{Table-as-Search (TaS)}, a structured planning framework that reformulates the InfoSeeking task as a Table Completion task. TaS maps each query into a structured table schema maintained in an external da

Refer to the [full paper](https://arxiv.org/abs/2602.06724v1) for detailed methodology.