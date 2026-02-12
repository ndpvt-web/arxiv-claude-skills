---
name: "hunt-instead-of-wait"
description: "The agency expected of Agentic Large Language Models goes beyond answering correctly, requiring autonomy to set goals and decide what to explore. Implements techniques from the paper 'Hunt Instead of Wait: Evaluating Deep Data Research on Large Language Models' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework), (database & query) or when the user references techniques from this research area."
---

# Hunt Instead of Wait: Evaluating Deep Data Research on Large Language Models

**Source:** [https://arxiv.org/abs/2602.02039v1](https://arxiv.org/abs/2602.02039v1)
**Category:** cs.AI | **Published:** 2026-02-02 | **Skill Score:** 77
**Authors:** Wei Liu, Peijie Yu, Michele Orini...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** deep data research (ddr)

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

> The agency expected of Agentic Large Language Models goes beyond answering correctly, requiring autonomy to set goals and decide what to explore. We term this investigatory intelligence, distinguishing it from executional intelligence, which merely completes assigned tasks. Data Science provides a natural testbed, as real-world analysis starts from raw data rather than explicit queries, yet few benchmarks focus on it. To address this, we introduce Deep Data Research (DDR), an open-ended task whe

Refer to the [full paper](https://arxiv.org/abs/2602.02039v1) for detailed methodology.