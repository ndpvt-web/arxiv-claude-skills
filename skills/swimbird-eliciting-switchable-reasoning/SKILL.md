---
name: "swimbird-eliciting-switchable-reasoning"
description: "Multimodal Large Language Models (MLLMs) have made remarkable progress in multimodal perception and reasoning by bridging vision and language. Implements techniques from the paper 'SwimBird: Eliciting Switchable Reasoning Mode in Hybrid Autoregressive MLLMs' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# SwimBird: Eliciting Switchable Reasoning Mode in Hybrid Autoregressive MLLMs

**Source:** [https://arxiv.org/abs/2602.06040v1](https://arxiv.org/abs/2602.06040v1)
**Category:** cs.CV | **Published:** 2026-02-05 | **Skill Score:** 62
**Authors:** Jintao Tong, Shilin Yan, Hongwei Xue...

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

> Multimodal Large Language Models (MLLMs) have made remarkable progress in multimodal perception and reasoning by bridging vision and language. However, most existing MLLMs perform reasoning primarily with textual CoT, which limits their effectiveness on vision-intensive tasks. Recent approaches inject a fixed number of continuous hidden states as "visual thoughts" into the reasoning process and improve visual performance, but often at the cost of degraded text-based logical reasoning. We argue t

Refer to the [full paper](https://arxiv.org/abs/2602.06040v1) for detailed methodology.