---
name: "addressing-corpus-knowledge-poisoning"
description: "Retrieval Augmented Generation (RAG) is a highly effective paradigm for keeping LLM-based responses up-to-date and reducing the likelihood of hallucinations. Implements techniques from the paper 'Addressing Corpus Knowledge Poisoning Attacks on RAG Using Sparse Attention' for extract, transform, and process data. Use when tasks involve (data processing), (search & retrieval) or when the user references techniques from this research area."
---

# Addressing Corpus Knowledge Poisoning Attacks on RAG Using Sparse Attention

**Source:** [https://arxiv.org/abs/2602.04711v2](https://arxiv.org/abs/2602.04711v2)
**Category:** cs.IR | **Published:** 2026-02-04 | **Skill Score:** 60
**Authors:** Sagie Dekel, Moshe Tennenholtz, Oren Kurland

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

## Research Context

> Retrieval Augmented Generation (RAG) is a highly effective paradigm for keeping LLM-based responses up-to-date and reducing the likelihood of hallucinations. Yet, RAG was recently shown to be quite vulnerable to corpus knowledge poisoning: an attacker injects misleading documents to the corpus to steer an LLM's output to an undesired response. We argue that the standard causal attention mechanism in LLMs enables harmful cross-document interactions, specifically in cases of attacks. Accordingly, 

Refer to the [full paper](https://arxiv.org/abs/2602.04711v2) for detailed methodology.