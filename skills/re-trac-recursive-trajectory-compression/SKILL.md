---
name: "re-trac-recursive-trajectory-compression"
description: "LLM-based deep research agents are largely built on the ReAct framework. Implements techniques from the paper 'RE-TRAC: REcursive TRAjectory Compression for Deep Search Agents' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# RE-TRAC: REcursive TRAjectory Compression for Deep Search Agents

**Source:** [https://arxiv.org/abs/2602.02486v1](https://arxiv.org/abs/2602.02486v1)
**Category:** cs.CL | **Published:** 2026-02-02 | **Skill Score:** 63
**Authors:** Jialiang Zhu, Gongrui Zhang, Xiaolong Ma...

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

> LLM-based deep research agents are largely built on the ReAct framework. This linear design makes it difficult to revisit earlier states, branch into alternative search directions, or maintain global awareness under long contexts, often leading to local optima, redundant exploration, and inefficient search. We propose Re-TRAC, an agentic framework that performs cross-trajectory exploration by generating a structured state representation after each trajectory to summarize evidence, uncertainties,

Refer to the [full paper](https://arxiv.org/abs/2602.02486v1) for detailed methodology.