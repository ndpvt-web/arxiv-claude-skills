---
name: "automated-benchmark-generation-from"
description: "Open-ended question answering (QA) evaluates a model's ability to perform contextualized reasoning beyond factual recall. Implements techniques from the paper 'Automated Benchmark Generation from Domain Guidelines Informed by Bloom's Taxonomy' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# Automated Benchmark Generation from Domain Guidelines Informed by Bloom's Taxonomy

**Source:** [https://arxiv.org/abs/2601.20253v1](https://arxiv.org/abs/2601.20253v1)
**Category:** cs.CL | **Published:** 2026-01-28 | **Skill Score:** 60
**Authors:** Si Chen, Le Huy Khiem, Annalisa Szymanski...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** a framework for automated benchmark generation from expert-authored guidelines informed by bloom's taxonomy

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

> Open-ended question answering (QA) evaluates a model's ability to perform contextualized reasoning beyond factual recall. This challenge is especially acute in practice-based domains, where knowledge is procedural and grounded in professional judgment, while most existing LLM benchmarks depend on pre-existing human exam datasets that are often unavailable in such settings. We introduce a framework for automated benchmark generation from expert-authored guidelines informed by Bloom's Taxonomy. It

Refer to the [full paper](https://arxiv.org/abs/2601.20253v1) for detailed methodology.