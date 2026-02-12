---
name: "sage-benchmarking-and-improving"
description: "Deep research agents have emerged as powerful systems for addressing complex queries. Implements techniques from the paper 'SAGE: Benchmarking and Improving Retrieval for Deep Research Agents' for extract, transform, and process data. Use when tasks involve (data processing), (search & retrieval), (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# SAGE: Benchmarking and Improving Retrieval for Deep Research Agents

**Source:** [https://arxiv.org/abs/2602.05975v2](https://arxiv.org/abs/2602.05975v2)
**Category:** cs.IR | **Published:** 2026-02-05 | **Skill Score:** 66
**Authors:** Tiansheng Hu, Yilun Zhao, Canyu Zhang...

## Core Capability

Extract, transform, and process data.

## Key Techniques

- **Retrieval-augmented** approach for grounding responses in external knowledge

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

> Deep research agents have emerged as powerful systems for addressing complex queries. Meanwhile, LLM-based retrievers have demonstrated strong capability in following instructions or reasoning. This raises a critical question: can LLM-based retrievers effectively contribute to deep research agent workflows? To investigate this, we introduce SAGE, a benchmark for scientific literature retrieval comprising 1,200 queries across four scientific domains, with a 200,000 paper retrieval corpus. We eval

Refer to the [full paper](https://arxiv.org/abs/2602.05975v2) for detailed methodology.