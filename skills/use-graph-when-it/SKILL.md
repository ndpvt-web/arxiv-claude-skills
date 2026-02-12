---
name: "use-graph-when-it"
description: "Large language models (LLMs) often struggle with knowledge-intensive tasks due to hallucinations and outdated parametric knowledge. Implements techniques from the paper 'Use Graph When It Needs: Efficiently and Adaptively Integrating Retrieval-Augmented Generation with Graphs' for extract, transform, and process data. Use when tasks involve (data processing), (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# Use Graph When It Needs: Efficiently and Adaptively Integrating Retrieval-Augmented Generation with Graphs

**Source:** [https://arxiv.org/abs/2602.03578v1](https://arxiv.org/abs/2602.03578v1)
**Category:** cs.CL | **Published:** 2026-02-03 | **Skill Score:** 75
**Authors:** Su Dong, Qinggang Zhang, Yilin Xiao...

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

> Large language models (LLMs) often struggle with knowledge-intensive tasks due to hallucinations and outdated parametric knowledge. While Retrieval-Augmented Generation (RAG) addresses this by integrating external corpora, its effectiveness is limited by fragmented information in unstructured domain documents. Graph-augmented RAG (GraphRAG) emerged to enhance contextual reasoning through structured knowledge graphs, yet paradoxically underperforms vanilla RAG in real-world scenarios, exhibiting 

Refer to the [full paper](https://arxiv.org/abs/2602.03578v1) for detailed methodology.