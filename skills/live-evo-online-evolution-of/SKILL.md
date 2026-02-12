---
name: "live-evo-online-evolution-of"
description: "Large language model (LLM) agents are increasingly equipped with memory, which are stored experience and reusable guidance that can improve task-solving performance. Implements techniques from the paper 'Live-Evo: Online Evolution of Agentic Memory from Continuous Feedback' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# Live-Evo: Online Evolution of Agentic Memory from Continuous Feedback

**Source:** [https://arxiv.org/abs/2602.02369v1](https://arxiv.org/abs/2602.02369v1)
**Category:** cs.AI | **Published:** 2026-02-02 | **Skill Score:** 75
**Authors:** Yaolun Zhang, Yiran Wu, Yijiong Yu...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** \textsc{live-evo}

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

> Large language model (LLM) agents are increasingly equipped with memory, which are stored experience and reusable guidance that can improve task-solving performance. Recent \emph{self-evolving} systems update memory based on interaction outcomes, but most existing evolution pipelines are developed for static train/test splits and only approximate online learning by folding static benchmarks, making them brittle under true distribution shift and continuous feedback. We introduce \textsc{Live-Evo}

Refer to the [full paper](https://arxiv.org/abs/2602.02369v1) for detailed methodology.