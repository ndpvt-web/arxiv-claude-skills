---
name: "agentxray-whiteboxing-agentic-systems"
description: "Large Language Models have shown strong capabilities in complex problem solving, yet many agentic systems remain difficult to interpret and control due to opaque internal workflows. Implements techniques from the paper 'AgentXRay: White-Boxing Agentic Systems via Workflow Reconstruction' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# AgentXRay: White-Boxing Agentic Systems via Workflow Reconstruction

**Source:** [https://arxiv.org/abs/2602.05353v2](https://arxiv.org/abs/2602.05353v2)
**Category:** cs.AI | **Published:** 2026-02-05 | **Skill Score:** 69
**Authors:** Ruijie Shi, Houbin Zhang, Yuecheng Han...

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

> Large Language Models have shown strong capabilities in complex problem solving, yet many agentic systems remain difficult to interpret and control due to opaque internal workflows. While some frameworks offer explicit architectures for collaboration, many deployed agentic systems operate as black boxes to users. We address this by introducing Agentic Workflow Reconstruction (AWR), a new task aiming to synthesize an explicit, interpretable stand-in workflow that approximates a black-box system u

Refer to the [full paper](https://arxiv.org/abs/2602.05353v2) for detailed methodology.