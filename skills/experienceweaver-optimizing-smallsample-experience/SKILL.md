---
name: "experienceweaver-optimizing-smallsample-experience"
description: "Clinical text improvement is vital for healthcare efficiency but remains difficult due to limited high-quality data and the complex constraints of medical documentation. Implements techniques from the paper 'ExperienceWeaver: Optimizing Small-sample Experience Learning for LLM-based Clinical Text Improvement' for extract, transform, and process data. Use when tasks involve (data processing), (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# ExperienceWeaver: Optimizing Small-sample Experience Learning for LLM-based Clinical Text Improvement

**Source:** [https://arxiv.org/abs/2602.00740v1](https://arxiv.org/abs/2602.00740v1)
**Category:** cs.CL | **Published:** 2026-01-31 | **Skill Score:** 80
**Authors:** Ziyan Xiao, Yinghao Zhu, Liang Peng...

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

> Clinical text improvement is vital for healthcare efficiency but remains difficult due to limited high-quality data and the complex constraints of medical documentation. While Large Language Models (LLMs) show promise, current approaches struggle in small-sample settings: supervised fine-tuning is data-intensive and costly, while retrieval-augmented generation often provides superficial corrections without capturing the reasoning behind revisions. To address these limitations, we propose Experie

Refer to the [full paper](https://arxiv.org/abs/2602.00740v1) for detailed methodology.