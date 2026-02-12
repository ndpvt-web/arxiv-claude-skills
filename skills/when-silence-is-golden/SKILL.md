---
name: "when-silence-is-golden"
description: "Large language models (LLMs) rarely admit uncertainty, often producing fluent but misleading answers, rather than abstaining (i.e., refusing to answer). Implements techniques from the paper 'When Silence Is Golden: Can LLMs Learn to Abstain in Temporal QA and Beyond?' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# When Silence Is Golden: Can LLMs Learn to Abstain in Temporal QA and Beyond?

**Source:** [https://arxiv.org/abs/2602.04755v1](https://arxiv.org/abs/2602.04755v1)
**Category:** cs.CL | **Published:** 2026-02-04 | **Skill Score:** 71
**Authors:** Xinyu Zhou, Chang Jin, Carsten Eickhoff...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** the first empirical study of training llms with an abstention ability while reasoning about temporal qa

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

> Large language models (LLMs) rarely admit uncertainty, often producing fluent but misleading answers, rather than abstaining (i.e., refusing to answer). This weakness is even evident in temporal question answering, where models frequently ignore time-sensitive evidence and conflate facts across different time-periods. In this paper, we present the first empirical study of training LLMs with an abstention ability while reasoning about temporal QA. Existing approaches such as calibration might be 

Refer to the [full paper](https://arxiv.org/abs/2602.04755v1) for detailed methodology.