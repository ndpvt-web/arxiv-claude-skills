---
name: "fat-cat-documentdriven-metacognitive-multiagent"
description: "The effectiveness of LLM-based agents is often limited not by model capacity alone, but by how efficiently contextual information is utilized at runtime. Implements techniques from the paper 'Fat-Cat: Document-Driven Metacognitive Multi-Agent System for Complex Reasoning' for extract, transform, and process data. Use when tasks involve (data processing), (search & retrieval), (agent framework), (design & ui) or when the user references techniques from this research area."
---

# Fat-Cat: Document-Driven Metacognitive Multi-Agent System for Complex Reasoning

**Source:** [https://arxiv.org/abs/2602.02206v2](https://arxiv.org/abs/2602.02206v2)
**Category:** cs.LG | **Published:** 2026-02-02 | **Skill Score:** 71
**Authors:** Tong Yang, Yemin Wang, Chaoning Zhang...

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

> The effectiveness of LLM-based agents is often limited not by model capacity alone, but by how efficiently contextual information is utilized at runtime. Existing agent frameworks rely on rigid, syntax-heavy state representations such as nested JSON, which require models to devote a substantial portion of their limited attention to syntactic processing rather than semantic reasoning. In this paper, we propose Fat-Cat, a document-driven agent architecture that improves the signal-to-noise ratio o

Refer to the [full paper](https://arxiv.org/abs/2602.02206v2) for detailed methodology.