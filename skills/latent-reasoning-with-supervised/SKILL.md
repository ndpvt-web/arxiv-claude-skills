---
name: "latent-reasoning-with-supervised"
description: "Reasoning with a chain-of-thought (CoT) enables Large Language Models (LLMs) to solve complex tasks but incurs significant inference costs due to the generation of long rationales. Implements techniques from the paper 'Latent Reasoning with Supervised Thinking States' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# Latent Reasoning with Supervised Thinking States

**Source:** [https://arxiv.org/abs/2602.08332v1](https://arxiv.org/abs/2602.08332v1)
**Category:** cs.CL | **Published:** 2026-02-09 | **Skill Score:** 59
**Authors:** Ido Amos, Avi Caciularu, Mor Geva...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** thinking states

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

> Reasoning with a chain-of-thought (CoT) enables Large Language Models (LLMs) to solve complex tasks but incurs significant inference costs due to the generation of long rationales. We propose Thinking States, a method that performs reasoning {\em while} the input is processing. Specifically, Thinking States generates sequences of thinking tokens every few input tokens, transforms the thoughts back into embedding space, and adds them to the following input tokens. This has two key advantages. Fir

Refer to the [full paper](https://arxiv.org/abs/2602.08332v1) for detailed methodology.