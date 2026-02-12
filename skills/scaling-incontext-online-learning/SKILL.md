---
name: "scaling-incontext-online-learning"
description: "Large language models (LLMs) achieve strong performance when all task-relevant information is available upfront, as in static prediction and instruction-following problems. Implements techniques from the paper 'Scaling In-Context Online Learning Capability of LLMs via Cross-Episode Meta-RL' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework), (prompt engineering), (security) or when the user references techniques from this research area."
---

# Scaling In-Context Online Learning Capability of LLMs via Cross-Episode Meta-RL

**Source:** [https://arxiv.org/abs/2602.04089v1](https://arxiv.org/abs/2602.04089v1)
**Category:** cs.AI | **Published:** 2026-02-03 | **Skill Score:** 97
**Authors:** Xiaofeng Lin, Sirou Zhu, Yilei Chen...

## Core Capability

Search, retrieve, and synthesize information.

## Workflow

1. Parse the user's information need or query
2. Formulate effective search strategies
3. Retrieve relevant documents, code, or data
4. Synthesize findings into a coherent response
5. Provide citations and references

## Security Considerations

- Check for OWASP Top 10 vulnerabilities
- Validate all user inputs at system boundaries
- Use parameterized queries for database access
- Follow the principle of least privilege

## Agent Coordination

- Decompose complex tasks into independent subtasks where possible
- Use parallel execution for independent subtasks
- Implement error recovery and retry logic
- Aggregate partial results when full completion fails

## Research Context

> Large language models (LLMs) achieve strong performance when all task-relevant information is available upfront, as in static prediction and instruction-following problems. However, many real-world decision-making tasks are inherently online: crucial information must be acquired through interaction, feedback is delayed, and effective behavior requires balancing information collection and exploitation over time. While in-context learning enables adaptation without weight updates, existing LLMs of

Refer to the [full paper](https://arxiv.org/abs/2602.04089v1) for detailed methodology.