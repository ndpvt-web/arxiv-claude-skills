---
name: "say-anything-but-this"
description: "Large language models (LLMs) reason over discrete token ID sequences, yet modern subword tokenizers routinely produce non-unique encodings: multiple token ID sequences can detokenize to identical s... Implements techniques from the paper 'Say Anything but This: When Tokenizer Betrays Reasoning in LLMs' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# Say Anything but This: When Tokenizer Betrays Reasoning in LLMs

**Source:** [https://arxiv.org/abs/2601.14658v1](https://arxiv.org/abs/2601.14658v1)
**Category:** cs.CL | **Published:** 2026-01-21 | **Skill Score:** 60
**Authors:** Navid Ayoobi, Marcus I Armstrong, Arjun Mukherjee

## Core Capability

Search, retrieve, and synthesize information.

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

> Large language models (LLMs) reason over discrete token ID sequences, yet modern subword tokenizers routinely produce non-unique encodings: multiple token ID sequences can detokenize to identical surface strings. This representational mismatch creates an unmeasured fragility wherein reasoning processes can fail. LLMs may treat two internal representations as distinct "words" even when they are semantically identical at the text level. In this work, we show that tokenization can betray LLM reason

Refer to the [full paper](https://arxiv.org/abs/2601.14658v1) for detailed methodology.