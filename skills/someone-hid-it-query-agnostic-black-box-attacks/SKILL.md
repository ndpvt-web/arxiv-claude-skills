---
name: "someone-hid-it-query-agnostic-black-box-attacks"
description: "Large language models (LLMs) have been serving as effective backbones for retrieval systems, including Retrieval-Augmentation-Generation (RAG), Dense Information Retriever (IR), and Agent Memory Re... Implements techniques from the paper '\"Someone Hid It\": Query-Agnostic Black-Box Attacks on LLM-Based Retrieval' for extract, transform, and process data. Use when tasks involve (data processing), (search & retrieval), (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# "Someone Hid It": Query-Agnostic Black-Box Attacks on LLM-Based Retrieval

**Source:** [https://arxiv.org/abs/2602.00364v2](https://arxiv.org/abs/2602.00364v2)
**Category:** cs.CR | **Published:** 2026-01-30 | **Skill Score:** 59
**Authors:** Jiate Li, Defu Cao, Li Li...

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

> Large language models (LLMs) have been serving as effective backbones for retrieval systems, including Retrieval-Augmentation-Generation (RAG), Dense Information Retriever (IR), and Agent Memory Retrieval. Recent studies have demonstrated that such LLM-based Retrieval (LLMR) is vulnerable to adversarial attacks, which manipulates documents by token-level injections and enables adversaries to either boost or diminish these documents in retrieval tasks. However, existing attack studies mainly (1) 

Refer to the [full paper](https://arxiv.org/abs/2602.00364v2) for detailed methodology.