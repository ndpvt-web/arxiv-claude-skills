---
name: "medspeak-a-knowledge-graphaided"
description: "Spoken question-answering (SQA) systems relying on automatic speech recognition (ASR) often struggle with accurately recognizing medical terminology. Implements techniques from the paper 'MedSpeak: A Knowledge Graph-Aided ASR Error Correction Framework for Spoken Medical QA' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# MedSpeak: A Knowledge Graph-Aided ASR Error Correction Framework for Spoken Medical QA

**Source:** [https://arxiv.org/abs/2602.00981v1](https://arxiv.org/abs/2602.00981v1)
**Category:** cs.CL | **Published:** 2026-02-01 | **Skill Score:** 79
**Authors:** Yutong Song, Shiva Shrestha, Chenhan Lyu...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Novel approach:** knowledge graph-aided asr error correction framework
- **Leverages:** both semantic relationships and phonetic information encoded in a medical knowledge graph

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

> Spoken question-answering (SQA) systems relying on automatic speech recognition (ASR) often struggle with accurately recognizing medical terminology. To this end, we propose MedSpeak, a novel knowledge graph-aided ASR error correction framework that refines noisy transcripts and improves downstream answer prediction by leveraging both semantic relationships and phonetic information encoded in a medical knowledge graph, together with the reasoning power of LLMs. Comprehensive experimental results

Refer to the [full paper](https://arxiv.org/abs/2602.00981v1) for detailed methodology.