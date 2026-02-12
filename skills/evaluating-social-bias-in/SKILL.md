---
name: "evaluating-social-bias-in"
description: "Social biases inherent in large language models (LLMs) raise significant fairness concerns. Implements techniques from the paper 'Evaluating Social Bias in RAG Systems: When External Context Helps and Reasoning Hurts' for extract, transform, and process data. Use when tasks involve (data processing), (search & retrieval), (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# Evaluating Social Bias in RAG Systems: When External Context Helps and Reasoning Hurts

**Source:** [https://arxiv.org/abs/2602.09442v1](https://arxiv.org/abs/2602.09442v1)
**Category:** cs.CL | **Published:** 2026-02-10 | **Skill Score:** 79
**Authors:** Shweta Parihar, Lu Cheng

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

> Social biases inherent in large language models (LLMs) raise significant fairness concerns. Retrieval-Augmented Generation (RAG) architectures, which retrieve external knowledge sources to enhance the generative capabilities of LLMs, remain susceptible to the same bias-related challenges. This work focuses on evaluating and understanding the social bias implications of RAG. Through extensive experiments across various retrieval corpora, LLMs, and bias evaluation datasets, encompassing more than 

Refer to the [full paper](https://arxiv.org/abs/2602.09442v1) for detailed methodology.