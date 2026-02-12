---
name: "swe-master-unleashing-the-potential"
description: "In this technical report, we present SWE-Master, an open-source and fully reproducible post-training framework for building effective software engineering agents. Implements techniques from the paper 'SWE-Master: Unleashing the Potential of Software Engineering Agents via Post-Training' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# SWE-Master: Unleashing the Potential of Software Engineering Agents via Post-Training

**Source:** [https://arxiv.org/abs/2602.03411v1](https://arxiv.org/abs/2602.03411v1)
**Category:** cs.SE | **Published:** 2026-02-03 | **Skill Score:** 90
**Authors:** Huatong Song, Lisheng Huang, Shuang Sun...

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

> In this technical report, we present SWE-Master, an open-source and fully reproducible post-training framework for building effective software engineering agents. SWE-Master systematically explores the complete agent development pipeline, including teacher-trajectory synthesis and data curation, long-horizon SFT, RL with real execution feedback, and inference framework design. Starting from an open-source base model with limited initial SWE capability, SWE-Master demonstrates how systematical op

Refer to the [full paper](https://arxiv.org/abs/2602.03411v1) for detailed methodology.