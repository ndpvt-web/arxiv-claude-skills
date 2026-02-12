---
name: "logicscore-finegrained-logic-evaluation"
description: "Current evaluation methods for Attributed Question Answering (AQA) suffer from \textit{attribution myopia}: they emphasize verification of isolated statements and their attributions but overlook th... Implements techniques from the paper 'LogicScore: Fine-grained Logic Evaluation of Conciseness, Completeness, and Determinateness in Attributed Question Answering' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# LogicScore: Fine-grained Logic Evaluation of Conciseness, Completeness, and Determinateness in Attributed Question Answering

**Source:** [https://arxiv.org/abs/2601.15050v3](https://arxiv.org/abs/2601.15050v3)
**Category:** cs.CL | **Published:** 2026-01-21 | **Skill Score:** 76
**Authors:** Zhichao Yan, Yunxiao Zhao, Jiapu Wang...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** \textsc{logicscore}

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

> Current evaluation methods for Attributed Question Answering (AQA) suffer from \textit{attribution myopia}: they emphasize verification of isolated statements and their attributions but overlook the global logical integrity of long-form answers. Consequently, Large Language Models (LLMs) often produce factually grounded yet logically incoherent responses with elusive deductive gaps. To mitigate this limitation, we present \textsc{LogicScore}, a unified evaluation framework that shifts the paradi

Refer to the [full paper](https://arxiv.org/abs/2601.15050v3) for detailed methodology.