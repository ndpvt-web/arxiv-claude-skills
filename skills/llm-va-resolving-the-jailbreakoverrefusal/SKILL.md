---
name: "llm-va-resolving-the-jailbreakoverrefusal"
description: "Safety-aligned LLMs suffer from two failure modes: jailbreak (answering harmful inputs) and over-refusal (declining benign queries). Implements techniques from the paper 'LLM-VA: Resolving the Jailbreak-Overrefusal Trade-off via Vector Alignment' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval) or when the user references techniques from this research area."
---

# LLM-VA: Resolving the Jailbreak-Overrefusal Trade-off via Vector Alignment

**Source:** [https://arxiv.org/abs/2601.19487v1](https://arxiv.org/abs/2601.19487v1)
**Category:** cs.LG | **Published:** 2026-01-27 | **Skill Score:** 59
**Authors:** Haonan Zhang, Dongxia Wang, Yi Liu...

## Core Capability

Search, retrieve, and synthesize information.

## Workflow

1. Parse the user's information need or query
2. Formulate effective search strategies
3. Retrieve relevant documents, code, or data
4. Synthesize findings into a coherent response
5. Provide citations and references

## Research Context

> Safety-aligned LLMs suffer from two failure modes: jailbreak (answering harmful inputs) and over-refusal (declining benign queries). Existing vector steering methods adjust the magnitude of answer vectors, but this creates a fundamental trade-off -- reducing jailbreak increases over-refusal and vice versa. We identify the root cause: LLMs encode the decision to answer (answer vector $v_a$) and the judgment of input safety (benign vector $v_b$) as nearly orthogonal directions, treating them as in

Refer to the [full paper](https://arxiv.org/abs/2601.19487v1) for detailed methodology.