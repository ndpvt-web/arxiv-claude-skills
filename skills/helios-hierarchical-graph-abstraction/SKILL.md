---
name: "helios-hierarchical-graph-abstraction"
description: "Large language models (LLMs) have recently been applied to binary decompilation, yet they still treat code as plain text and ignore the graphs that govern program control flow. Implements techniques from the paper 'HELIOS: Hierarchical Graph Abstraction for Structure-Aware LLM Decompilation' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework), (prompt engineering), (security) or when the user references techniques from this research area."
---

# HELIOS: Hierarchical Graph Abstraction for Structure-Aware LLM Decompilation

**Source:** [https://arxiv.org/abs/2601.14598v2](https://arxiv.org/abs/2601.14598v2)
**Category:** cs.SE | **Published:** 2026-01-21 | **Skill Score:** 75
**Authors:** Yonatan Gizachew Achamyeleh, Harsh Thomare, Mohammad Abdullah Al Faruque

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** \textsc{helios}

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

> Large language models (LLMs) have recently been applied to binary decompilation, yet they still treat code as plain text and ignore the graphs that govern program control flow. This limitation often yields syntactically fragile and logically inconsistent output, especially for optimized binaries. This paper presents \textsc{HELIOS}, a framework that reframes LLM-based decompilation as a structured reasoning task. \textsc{HELIOS} summarizes a binary's control flow and function calls into a hierar

Refer to the [full paper](https://arxiv.org/abs/2601.14598v2) for detailed methodology.