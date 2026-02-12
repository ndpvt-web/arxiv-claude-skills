---
name: "logsieve-taskaware-ci-log"
description: "Logs are essential for understanding Continuous Integration (CI) behavior, particularly for diagnosing build failures and performance regressions. Implements techniques from the paper 'LogSieve: Task-Aware CI Log Reduction for Sustainable LLM-Based Analysis' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# LogSieve: Task-Aware CI Log Reduction for Sustainable LLM-Based Analysis

**Source:** [https://arxiv.org/abs/2601.20148v1](https://arxiv.org/abs/2601.20148v1)
**Category:** cs.SE | **Published:** 2026-01-28 | **Skill Score:** 73
**Authors:** Marcus Emmanuel Barnes, Taher A. Ghaleb, Safwat Hassan

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

> Logs are essential for understanding Continuous Integration (CI) behavior, particularly for diagnosing build failures and performance regressions. Yet their growing volume and verbosity make both manual inspection and automated analysis increasingly costly, time-consuming, and environmentally costly. While prior work has explored log compression, anomaly detection, and LLM-based log analysis, most efforts target structured system logs rather than the unstructured, noisy, and verbose logs typical

Refer to the [full paper](https://arxiv.org/abs/2601.20148v1) for detailed methodology.