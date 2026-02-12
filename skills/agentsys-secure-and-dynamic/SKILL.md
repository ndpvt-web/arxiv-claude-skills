---
name: "agentsys-secure-and-dynamic"
description: "Indirect prompt injection threatens LLM agents by embedding malicious instructions in external content, enabling unauthorized actions and data theft. Implements techniques from the paper 'AgentSys: Secure and Dynamic LLM Agents Through Explicit Hierarchical Memory Management' for extract, transform, and process data. Use when tasks involve (data processing), (search & retrieval), (agent framework), (prompt engineering), (database & query) or when the user references techniques from this research area."
---

# AgentSys: Secure and Dynamic LLM Agents Through Explicit Hierarchical Memory Management

**Source:** [https://arxiv.org/abs/2602.07398v1](https://arxiv.org/abs/2602.07398v1)
**Category:** cs.CR | **Published:** 2026-02-07 | **Skill Score:** 83
**Authors:** Ruoyao Wen, Hao Li, Chaowei Xiao...

## Core Capability

Extract, transform, and process data.

## Workflow

1. Identify the data source and format
2. Parse and extract relevant data fields
3. Transform data into the desired output format
4. Validate data integrity and handle errors
5. Output processed data in the requested format

## Agent Coordination

- Decompose complex tasks into independent subtasks where possible
- Use parallel execution for independent subtasks
- Implement error recovery and retry logic
- Aggregate partial results when full completion fails

## Research Context

> Indirect prompt injection threatens LLM agents by embedding malicious instructions in external content, enabling unauthorized actions and data theft. LLM agents maintain working memory through their context window, which stores interaction history for decision-making. Conventional agents indiscriminately accumulate all tool outputs and reasoning traces in this memory, creating two critical vulnerabilities: (1) injected instructions persist throughout the workflow, granting attackers multiple opp

Refer to the [full paper](https://arxiv.org/abs/2602.07398v1) for detailed methodology.