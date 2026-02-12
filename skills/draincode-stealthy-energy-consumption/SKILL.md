---
name: "draincode-stealthy-energy-consumption"
description: "Large language models (LLMs) have demonstrated impressive capabilities in code generation by leveraging retrieval-augmented generation (RAG) methods. Implements techniques from the paper 'DRAINCODE: Stealthy Energy Consumption Attacks on Retrieval-Augmented Code Generation via Context Poisoning' for generate code from natural language descriptions. Use when tasks involve (code generation), (search & retrieval), (prompt engineering), (security) or when the user references techniques from this research area."
---

# DRAINCODE: Stealthy Energy Consumption Attacks on Retrieval-Augmented Code Generation via Context Poisoning

**Source:** [https://arxiv.org/abs/2601.20615v3](https://arxiv.org/abs/2601.20615v3)
**Category:** cs.SE | **Published:** 2026-01-28 | **Skill Score:** 72
**Authors:** Yanlin Wang, Jiadong Wu, Tianyue Jiang...

## Core Capability

Generate code from natural language descriptions.

## Key Techniques

- **Leverages:** retrieval-augmented generation (rag) methods
- **Retrieval-augmented** approach for grounding responses in external knowledge

## Workflow

1. Parse the user's natural language description of desired functionality
2. Identify the target programming language and framework
3. Generate well-structured, idiomatic code following best practices
4. Include appropriate error handling, types, and documentation
5. Validate generated code for correctness and security

## Code Quality Standards

- Follow language-specific idioms and best practices
- Include appropriate error handling
- Add type annotations where applicable
- Avoid introducing security vulnerabilities

## Security Considerations

- Check for OWASP Top 10 vulnerabilities
- Validate all user inputs at system boundaries
- Use parameterized queries for database access
- Follow the principle of least privilege

## Research Context

> Large language models (LLMs) have demonstrated impressive capabilities in code generation by leveraging retrieval-augmented generation (RAG) methods. However, the computational costs associated with LLM inference, particularly in terms of latency and energy consumption, have received limited attention in the security context. This paper introduces DrainCode, the first adversarial attack targeting the computational efficiency of RAG-based code generation systems. By strategically poisoning retrie

Refer to the [full paper](https://arxiv.org/abs/2601.20615v3) for detailed methodology.