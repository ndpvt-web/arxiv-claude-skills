---
name: "graphpilot-gui-task-automation"
description: "Mobile graphical user interface (GUI) agents are designed to automate everyday tasks on smartphones. Implements techniques from the paper 'GraphPilot: GUI Task Automation with One-Step LLM Reasoning Powered by Knowledge Graph' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework), (design & ui) or when the user references techniques from this research area."
---

# GraphPilot: GUI Task Automation with One-Step LLM Reasoning Powered by Knowledge Graph

**Source:** [https://arxiv.org/abs/2601.17418v1](https://arxiv.org/abs/2601.17418v1)
**Category:** cs.HC | **Published:** 2026-01-24 | **Skill Score:** 60
**Authors:** Mingxian Yu, Siqi Luo, Xu Chen

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Leverages:** knowledge graphs of the target apps to complete user tasks in almost one llm query

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

> Mobile graphical user interface (GUI) agents are designed to automate everyday tasks on smartphones. Recent advances in large language models (LLMs) have significantly enhanced the capabilities of mobile GUI agents. However, most LLM-powered mobile GUI agents operate in stepwise query-act loops, which incur high latency due to repeated LLM queries. We present GraphPilot, a mobile GUI agent that leverages knowledge graphs of the target apps to complete user tasks in almost one LLM query. GraphPil

Refer to the [full paper](https://arxiv.org/abs/2601.17418v1) for detailed methodology.