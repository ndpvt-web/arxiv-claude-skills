---
name: "beyond-texttosql-can-llms"
description: "SQL is central to enterprise data engineering, yet generating fully correct SQL code in a single attempt remains difficult, even for experienced developers and advanced text-to-SQL LLMs, often requ... Implements techniques from the paper 'Beyond Text-to-SQL: Can LLMs Really Debug Enterprise ETL SQL?' for extract, transform, and process data. Use when tasks involve (data processing), (search & retrieval), (agent framework), (database & query) or when the user references techniques from this research area."
---

# Beyond Text-to-SQL: Can LLMs Really Debug Enterprise ETL SQL?

**Source:** [https://arxiv.org/abs/2601.18119v1](https://arxiv.org/abs/2601.18119v1)
**Category:** cs.AI | **Published:** 2026-01-26 | **Skill Score:** 64
**Authors:** Jing Ye, Yiwen Duan, Yonghong Yu...

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

> SQL is central to enterprise data engineering, yet generating fully correct SQL code in a single attempt remains difficult, even for experienced developers and advanced text-to-SQL LLMs, often requiring multiple debugging iterations. We introduce OurBench, the first benchmark for enterprise-level SQL reasoning and debugging. Our benchmark is built on two key innovations: (1) an automated construction workflow that uses reverse engineering to systematically inject realistic bugs into large-scale 

Refer to the [full paper](https://arxiv.org/abs/2601.18119v1) for detailed methodology.