---
name: "rgfl-reasoning-guided-fault"
description: "Fault Localization (FL) is a critical step in Automated Program Repair (APR), and its importance has increased with the rise of Large Language Model (LLM)-based repair agents. Implements techniques from the paper 'RGFL: Reasoning Guided Fault Localization for Automated Program Repair Using Large Language Models' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# RGFL: Reasoning Guided Fault Localization for Automated Program Repair Using Large Language Models

**Source:** [https://arxiv.org/abs/2601.18044v1](https://arxiv.org/abs/2601.18044v1)
**Category:** cs.SE | **Published:** 2026-01-25 | **Skill Score:** 59
**Authors:** Melika Sepidband, Hamed Taherkhani, Hung Viet Pham...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** a novel project-level fl approach that improves
- **Novel approach:** project-level fl approach
- **Achievement:** current llm context limits

## Workflow

1. Parse the user's information need or query
2. Formulate effective search strategies
3. Retrieve relevant documents, code, or data
4. Synthesize findings into a coherent response
5. Provide citations and references

## Agent Coordination

- Decompose complex tasks into independent subtasks where possible
- Use parallel execution for independent subtasks
- Implement error recovery and retry logic
- Aggregate partial results when full completion fails

## Research Context

> Fault Localization (FL) is a critical step in Automated Program Repair (APR), and its importance has increased with the rise of Large Language Model (LLM)-based repair agents. In realistic project-level repair scenarios, software repositories often span millions of tokens, far exceeding current LLM context limits. Consequently, models must first identify a small, relevant subset of code, making accurate FL essential for effective repair. We present a novel project-level FL approach that improves

Refer to the [full paper](https://arxiv.org/abs/2601.18044v1) for detailed methodology.