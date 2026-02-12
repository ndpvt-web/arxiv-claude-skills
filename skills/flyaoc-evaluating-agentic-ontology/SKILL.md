---
name: "flyaoc-evaluating-agentic-ontology"
description: "Scientific knowledge bases accelerate discovery by curating findings from primary literature into structured, queryable formats for both human researchers and emerging AI systems. Implements techniques from the paper 'FlyAOC: Evaluating Agentic Ontology Curation of Drosophila Scientific Knowledge Bases' for extract, transform, and process data. Use when tasks involve (data processing), (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# FlyAOC: Evaluating Agentic Ontology Curation of Drosophila Scientific Knowledge Bases

**Source:** [https://arxiv.org/abs/2602.09163v1](https://arxiv.org/abs/2602.09163v1)
**Category:** cs.AI | **Published:** 2026-02-09 | **Skill Score:** 84
**Authors:** Xingjian Zhang, Sophia Moylan, Ziyang Xiong...

## Core Capability

Extract, transform, and process data.

## Key Techniques

- **Proposed technique:** flybench to

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

> Scientific knowledge bases accelerate discovery by curating findings from primary literature into structured, queryable formats for both human researchers and emerging AI systems. Maintaining these resources requires expert curators to search relevant papers, reconcile evidence across documents, and produce ontology-grounded annotations - a workflow that existing benchmarks, focused on isolated subtasks like named entity recognition or relation extraction, do not capture. We present FlyBench to 

Refer to the [full paper](https://arxiv.org/abs/2602.09163v1) for detailed methodology.