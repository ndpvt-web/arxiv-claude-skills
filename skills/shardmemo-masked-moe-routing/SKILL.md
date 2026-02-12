---
name: "shardmemo-masked-moe-routing"
description: "Agentic large language model (LLM) systems rely on external memory for long-horizon state and concurrent multi-agent execution, but centralized indexes and heuristic partitions become bottlenecks a... Implements techniques from the paper 'ShardMemo: Masked MoE Routing for Sharded Agentic LLM Memory' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# ShardMemo: Masked MoE Routing for Sharded Agentic LLM Memory

**Source:** [https://arxiv.org/abs/2601.21545v1](https://arxiv.org/abs/2601.21545v1)
**Category:** cs.AI | **Published:** 2026-01-29 | **Skill Score:** 67
**Authors:** Yang Zhao, Chengxiao Dai, Yue Xiu...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Multi-agent architecture** for task decomposition and parallel execution

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

> Agentic large language model (LLM) systems rely on external memory for long-horizon state and concurrent multi-agent execution, but centralized indexes and heuristic partitions become bottlenecks as memory volume and parallel access grow. We present ShardMemo, a budgeted tiered memory service with Tier A per-agent working state, Tier B sharded evidence with shard-local approximate nearest neighbor (ANN) indexes, and Tier C, a versioned skill library. Tier B enforces scope-before-routing: structu

Refer to the [full paper](https://arxiv.org/abs/2601.21545v1) for detailed methodology.