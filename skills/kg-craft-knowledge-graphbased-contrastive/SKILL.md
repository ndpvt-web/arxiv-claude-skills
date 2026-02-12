---
name: "kg-craft-knowledge-graphbased-contrastive"
description: "Claim verification is a core component of automated fact-checking systems, aimed at determining the truthfulness of a statement by assessing it against reliable evidence sources such as documents o... Implements techniques from the paper 'KG-CRAFT: Knowledge Graph-based Contrastive Reasoning with LLMs for Enhancing Automated Fact-checking' for extract, transform, and process data. Use when tasks involve (data processing), (search & retrieval), (agent framework), (design & ui) or when the user references techniques from this research area."
---

# KG-CRAFT: Knowledge Graph-based Contrastive Reasoning with LLMs for Enhancing Automated Fact-checking

**Source:** [https://arxiv.org/abs/2601.19447v1](https://arxiv.org/abs/2601.19447v1)
**Category:** cs.CL | **Published:** 2026-01-27 | **Skill Score:** 76
**Authors:** Vítor N. Lourenço, Aline Paes, Tillman Weyde...

## Core Capability

Extract, transform, and process data.

## Key Techniques

- **Leverages:** large language models (llms) augmented with contrastive questions grounded in a knowledge graph

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

> Claim verification is a core component of automated fact-checking systems, aimed at determining the truthfulness of a statement by assessing it against reliable evidence sources such as documents or knowledge bases. This work presents KG-CRAFT, a method that improves automatic claim verification by leveraging large language models (LLMs) augmented with contrastive questions grounded in a knowledge graph. KG-CRAFT first constructs a knowledge graph from claims and associated reports, then formula

Refer to the [full paper](https://arxiv.org/abs/2601.19447v1) for detailed methodology.