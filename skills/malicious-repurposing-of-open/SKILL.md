---
name: "malicious-repurposing-of-open"
description: "The rapid evolution of large language models (LLMs) has fuelled enthusiasm about their role in advancing scientific discovery, with studies exploring LLMs that autonomously generate and evaluate no... Implements techniques from the paper 'Malicious Repurposing of Open Science Artefacts by Using Large Language Models' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework), (security) or when the user references techniques from this research area."
---

# Malicious Repurposing of Open Science Artefacts by Using Large Language Models

**Source:** [https://arxiv.org/abs/2601.18998v1](https://arxiv.org/abs/2601.18998v1)
**Category:** cs.CL | **Published:** 2026-01-26 | **Skill Score:** 69
**Authors:** Zahra Hashemi, Zhiqiang Zhong, Jun Pang...

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

> The rapid evolution of large language models (LLMs) has fuelled enthusiasm about their role in advancing scientific discovery, with studies exploring LLMs that autonomously generate and evaluate novel research ideas. However, little attention has been given to the possibility that such models could be exploited to produce harmful research by repurposing open science artefacts for malicious ends. We fill the gap by introducing an end-to-end pipeline that first bypasses LLM safeguards through pers

Refer to the [full paper](https://arxiv.org/abs/2601.18998v1) for detailed methodology.