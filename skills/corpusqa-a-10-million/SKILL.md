---
name: "corpusqa-a-10-million"
description: "While large language models now handle million-token contexts, their capacity for reasoning across entire document repositories remains largely untested. Implements techniques from the paper 'CorpusQA: A 10 Million Token Benchmark for Corpus-Level Analysis and Reasoning' for extract, transform, and process data. Use when tasks involve (data processing), (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# CorpusQA: A 10 Million Token Benchmark for Corpus-Level Analysis and Reasoning

**Source:** [https://arxiv.org/abs/2601.14952v1](https://arxiv.org/abs/2601.14952v1)
**Category:** cs.CL | **Published:** 2026-01-21 | **Skill Score:** 98
**Authors:** Zhiyuan Lu, Chenliang Li, Yingcheng Shi...

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

> While large language models now handle million-token contexts, their capacity for reasoning across entire document repositories remains largely untested. Existing benchmarks are inadequate, as they are mostly limited to single long texts or rely on a "sparse retrieval" assumption-that answers can be derived from a few relevant chunks. This assumption fails for true corpus-level analysis, where evidence is highly dispersed across hundreds of documents and answers require global integration, compa

Refer to the [full paper](https://arxiv.org/abs/2601.14952v1) for detailed methodology.