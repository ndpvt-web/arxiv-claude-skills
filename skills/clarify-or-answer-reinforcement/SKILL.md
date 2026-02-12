---
name: "clarify-or-answer-reinforcement"
description: "Real-world visual question answering (VQA) is often context-dependent: an image-question pair may be under-specified, such that the correct answer depends on external information that is not observ... Implements techniques from the paper 'Clarify or Answer: Reinforcement Learning for Agentic VQA with Context Under-specification' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# Clarify or Answer: Reinforcement Learning for Agentic VQA with Context Under-specification

**Source:** [https://arxiv.org/abs/2601.16400v1](https://arxiv.org/abs/2601.16400v1)
**Category:** cs.CL | **Published:** 2026-01-23 | **Skill Score:** 61
**Authors:** Zongwan Cao, Bingbing Wen, Lucy Lu Wang

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** coa(clarify-or-answer)

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

> Real-world visual question answering (VQA) is often context-dependent: an image-question pair may be under-specified, such that the correct answer depends on external information that is not observable in the image. In such cases, directly answering can lead to confident but incorrect predictions. We propose CoA(Clarify-or-Answer), an ask-or-answer agent that separately models the decision to ask or answer, and what to ask if needed. CoA first determines whether clarification is necessary; if so

Refer to the [full paper](https://arxiv.org/abs/2601.16400v1) for detailed methodology.