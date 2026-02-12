---
name: "infa-guard-mitigating-malicious-propagation"
description: "The rapid advancement of Large Language Model (LLM)-based Multi-Agent Systems (MAS) has introduced significant security vulnerabilities, where malicious influence can propagate virally through inte... Implements techniques from the paper 'INFA-Guard: Mitigating Malicious Propagation via Infection-Aware Safeguarding in LLM-Based Multi-Agent Systems' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework), (security) or when the user references techniques from this research area."
---

# INFA-Guard: Mitigating Malicious Propagation via Infection-Aware Safeguarding in LLM-Based Multi-Agent Systems

**Source:** [https://arxiv.org/abs/2601.14667v1](https://arxiv.org/abs/2601.14667v1)
**Category:** cs.MA | **Published:** 2026-01-21 | **Skill Score:** 67
**Authors:** Yijin Zhou, Xiaoya Lu, Dongrui Liu...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** infection-aware guard
- **Multi-agent architecture** for task decomposition and parallel execution

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

> The rapid advancement of Large Language Model (LLM)-based Multi-Agent Systems (MAS) has introduced significant security vulnerabilities, where malicious influence can propagate virally through inter-agent communication. Conventional safeguards often rely on a binary paradigm that strictly distinguishes between benign and attack agents, failing to account for infected agents i.e., benign entities converted by attack agents. In this paper, we propose Infection-Aware Guard, INFA-Guard, a novel defe

Refer to the [full paper](https://arxiv.org/abs/2601.14667v1) for detailed methodology.