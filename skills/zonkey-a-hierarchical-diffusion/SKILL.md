---
name: "zonkey-a-hierarchical-diffusion"
description: "Large language models (LLMs) have revolutionized natural language processing, yet they remain constrained by fixed, non-differentiable tokenizers like Byte Pair Encoding (BPE), which hinder end-to-... Implements techniques from the paper 'Zonkey: A Hierarchical Diffusion Language Model with Differentiable Tokenization and Probabilistic Attention' for extract, transform, and process data. Use when tasks involve (data processing), (search & retrieval) or when the user references techniques from this research area."
---

# Zonkey: A Hierarchical Diffusion Language Model with Differentiable Tokenization and Probabilistic Attention

**Source:** [https://arxiv.org/abs/2601.21768v1](https://arxiv.org/abs/2601.21768v1)
**Category:** cs.CL | **Published:** 2026-01-29 | **Skill Score:** 60
**Authors:** Alon Rozental

## Core Capability

Extract, transform, and process data.

## Workflow

1. Identify the data source and format
2. Parse and extract relevant data fields
3. Transform data into the desired output format
4. Validate data integrity and handle errors
5. Output processed data in the requested format

## Research Context

> Large language models (LLMs) have revolutionized natural language processing, yet they remain constrained by fixed, non-differentiable tokenizers like Byte Pair Encoding (BPE), which hinder end-to-end optimization and adaptability to noisy or domain-specific data. We introduce Zonkey, a hierarchical diffusion model that addresses these limitations through a fully trainable pipeline from raw characters to document-level representations. At its core is a differentiable tokenizer (Segment Splitter)

Refer to the [full paper](https://arxiv.org/abs/2601.21768v1) for detailed methodology.