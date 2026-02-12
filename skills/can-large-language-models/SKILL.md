---
name: "can-large-language-models"
description: "Large language models (LLMs) are trained and tested extensively on symbolic representations such as code and graphs, yet real-world user tasks are often specified in natural language. Implements techniques from the paper 'Can Large Language Models Generalize Procedures Across Representations?' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# Can Large Language Models Generalize Procedures Across Representations?

**Source:** [https://arxiv.org/abs/2602.03542v1](https://arxiv.org/abs/2602.03542v1)
**Category:** cs.CL | **Published:** 2026-02-03 | **Skill Score:** 59
**Authors:** Fangru Lin, Valentin Hofmann, Xingchen Wan...

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

> Large language models (LLMs) are trained and tested extensively on symbolic representations such as code and graphs, yet real-world user tasks are often specified in natural language. To what extent can LLMs generalize across these representations? Here, we approach this question by studying isomorphic tasks involving procedures represented in code, graphs, and natural language (e.g., scheduling steps in planning). We find that training LLMs with popular post-training methods on graphs or code d

Refer to the [full paper](https://arxiv.org/abs/2602.03542v1) for detailed methodology.