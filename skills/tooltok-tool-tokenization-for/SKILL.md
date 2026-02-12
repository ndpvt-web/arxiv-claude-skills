---
name: "tooltok-tool-tokenization-for"
description: "Existing GUI agent models relying on coordinate-based one-step visual grounding struggle with generalizing to varying input resolutions and aspect ratios. Implements techniques from the paper 'ToolTok: Tool Tokenization for Efficient and Generalizable GUI Agents' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# ToolTok: Tool Tokenization for Efficient and Generalizable GUI Agents

**Source:** [https://arxiv.org/abs/2602.02548v1](https://arxiv.org/abs/2602.02548v1)
**Category:** cs.LG | **Published:** 2026-01-30 | **Skill Score:** 91
**Authors:** Xiaoce Wang, Guibin Zhang, Junzhe Li...

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

> Existing GUI agent models relying on coordinate-based one-step visual grounding struggle with generalizing to varying input resolutions and aspect ratios. Alternatives introduce coordinate-free strategies yet suffer from learning under severe data scarcity. To address the limitations, we propose ToolTok, a novel paradigm of multi-step pathfinding for GUI agents, where operations are modeled as a sequence of progressive tool usage. Specifically, we devise tools aligned with human interaction habi

Refer to the [full paper](https://arxiv.org/abs/2602.02548v1) for detailed methodology.