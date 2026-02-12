---
name: "fs-researcher-testtime-scaling-for"
description: "Deep research is emerging as a representative long-horizon task for large language model (LLM) agents. Implements techniques from the paper 'FS-Researcher: Test-Time Scaling for Long-Horizon Research Tasks with File-System-Based Agents' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# FS-Researcher: Test-Time Scaling for Long-Horizon Research Tasks with File-System-Based Agents

**Source:** [https://arxiv.org/abs/2602.01566v1](https://arxiv.org/abs/2602.01566v1)
**Category:** cs.CL | **Published:** 2026-02-02 | **Skill Score:** 72
**Authors:** Chiwei Zhu, Benfeng Xu, Mingxuan Du...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** fs-researcher
- **Achievement:** model context limits

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

> Deep research is emerging as a representative long-horizon task for large language model (LLM) agents. However, long trajectories in deep research often exceed model context limits, compressing token budgets for both evidence collection and report writing, and preventing effective test-time scaling. We introduce FS-Researcher, a file-system-based, dual-agent framework that scales deep research beyond the context window via a persistent workspace. Specifically, a Context Builder agent acts as a l

Refer to the [full paper](https://arxiv.org/abs/2602.01566v1) for detailed methodology.