---
name: "ewsjf-an-adaptive-scheduler"
description: "Serving Large Language Models (LLMs) under mixed workloads--short, latency-sensitive interactive queries alongside long, throughput-oriented batch requests--poses a fundamental scheduling challenge. Implements techniques from the paper 'EWSJF: An Adaptive Scheduler with Hybrid Partitioning for Mixed-Workload LLM Inference' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (design & ui) or when the user references techniques from this research area."
---

# EWSJF: An Adaptive Scheduler with Hybrid Partitioning for Mixed-Workload LLM Inference

**Source:** [https://arxiv.org/abs/2601.21758v1](https://arxiv.org/abs/2601.21758v1)
**Category:** cs.DC | **Published:** 2026-01-29 | **Skill Score:** 59
**Authors:** Bronislav Sidik, Chaya Levi, Joseph Kampeas

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** ewsjf (effective workload-based shortest job first)

## Workflow

1. Parse the user's information need or query
2. Formulate effective search strategies
3. Retrieve relevant documents, code, or data
4. Synthesize findings into a coherent response
5. Provide citations and references

## Research Context

> Serving Large Language Models (LLMs) under mixed workloads--short, latency-sensitive interactive queries alongside long, throughput-oriented batch requests--poses a fundamental scheduling challenge. Standard First-Come, First-Served (FCFS) policies suffer from severe head-of-line blocking, leading to high tail latency and underutilized hardware. We introduce EWSJF (Effective Workload-based Shortest Job First), an adaptive request-level scheduler that learns workload structure in real time to joi

Refer to the [full paper](https://arxiv.org/abs/2601.21758v1) for detailed methodology.