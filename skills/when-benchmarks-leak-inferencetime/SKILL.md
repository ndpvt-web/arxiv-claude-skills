---
name: "when-benchmarks-leak-inferencetime"
description: "Benchmark-based evaluation is the de facto standard for comparing large language models (LLMs). Implements techniques from the paper 'When Benchmarks Leak: Inference-Time Decontamination for LLMs' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval) or when the user references techniques from this research area."
---

# When Benchmarks Leak: Inference-Time Decontamination for LLMs

**Source:** [https://arxiv.org/abs/2601.19334v1](https://arxiv.org/abs/2601.19334v1)
**Category:** cs.CL | **Published:** 2026-01-27 | **Skill Score:** 61
**Authors:** Jianzhe Chai, Yu Zhe, Jun Sakuma

## Core Capability

Search, retrieve, and synthesize information.

## Workflow

1. Parse the user's information need or query
2. Formulate effective search strategies
3. Retrieve relevant documents, code, or data
4. Synthesize findings into a coherent response
5. Provide citations and references

## Research Context

> Benchmark-based evaluation is the de facto standard for comparing large language models (LLMs). However, its reliability is increasingly threatened by test set contamination, where test samples or their close variants leak into training data and artificially inflate reported performance. To address this issue, prior work has explored two main lines of mitigation. One line attempts to identify and remove contaminated benchmark items before evaluation, but this inevitably alters the evaluation set

Refer to the [full paper](https://arxiv.org/abs/2601.19334v1) for detailed methodology.