---
name: "linguamap-which-layers-of"
description: "Despite multilingual pretraining, large language models often struggle with non-English tasks, particularly in language control, the ability to respond in the intended language. Implements techniques from the paper 'LinguaMap: Which Layers of LLMs Speak Your Language and How to Tune Them?' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# LinguaMap: Which Layers of LLMs Speak Your Language and How to Tune Them?

**Source:** [https://arxiv.org/abs/2601.20009v1](https://arxiv.org/abs/2601.20009v1)
**Category:** cs.CL | **Published:** 2026-01-27 | **Skill Score:** 83
**Authors:** J. Ben Tamo, Daniel Carlander-Reuterfelt, Jonathan Rubin...

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

> Despite multilingual pretraining, large language models often struggle with non-English tasks, particularly in language control, the ability to respond in the intended language. We identify and characterize two key failure modes: the multilingual transfer bottleneck (correct language, incorrect task response) and the language consistency bottleneck (correct task response, wrong language). To systematically surface these issues, we design a four-scenario evaluation protocol spanning MMLU, MGSM, a

Refer to the [full paper](https://arxiv.org/abs/2601.20009v1) for detailed methodology.