---
name: "vihermes-a-graphgrounded-multihop"
description: "Question Answering (QA) over regulatory documents is inherently challenging due to the need for multihop reasoning across legally interdependent texts, a requirement that is particularly pronounced... Implements techniques from the paper 'ViHERMES: A Graph-Grounded Multihop Question Answering Benchmark and System for Vietnamese Healthcare Regulations' for extract, transform, and process data. Use when tasks involve (data processing), (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# ViHERMES: A Graph-Grounded Multihop Question Answering Benchmark and System for Vietnamese Healthcare Regulations

**Source:** [https://arxiv.org/abs/2602.07361v1](https://arxiv.org/abs/2602.07361v1)
**Category:** cs.CL | **Published:** 2026-02-07 | **Skill Score:** 97
**Authors:** Long S. T. Nguyen, Quan M. Bui, Tin T. Ngo...

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

> Question Answering (QA) over regulatory documents is inherently challenging due to the need for multihop reasoning across legally interdependent texts, a requirement that is particularly pronounced in the healthcare domain where regulations are hierarchically structured and frequently revised through amendments and cross-references. Despite recent progress in retrieval-augmented and graph-based QA methods, systematic evaluation in this setting remains limited, especially for low-resource languag

Refer to the [full paper](https://arxiv.org/abs/2602.07361v1) for detailed methodology.