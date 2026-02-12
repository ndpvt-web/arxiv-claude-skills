---
name: "protecting-context-and-prompts"
description: "Large Language Model (LLM) applications are vulnerable to prompt injection and context manipulation attacks that traditional security models cannot prevent. Implements techniques from the paper 'Protecting Context and Prompts: Deterministic Security for Non-Deterministic AI' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework), (prompt engineering), (security) or when the user references techniques from this research area."
---

# Protecting Context and Prompts: Deterministic Security for Non-Deterministic AI

**Source:** [https://arxiv.org/abs/2602.10481v1](https://arxiv.org/abs/2602.10481v1)
**Category:** cs.CR | **Published:** 2026-02-11 | **Skill Score:** 65
**Authors:** Mohan Rajagopalan, Vinay Rao

## Core Capability

Build and orchestrate AI agent workflows.

## Key Techniques

- **Proposed technique:** two novel primitives--authenticated prompts and authenticated context--that provide cryptographically verifiable provenance across llm workflows

## Workflow

1. Decompose complex tasks into subtasks
2. Plan agent coordination and communication patterns
3. Execute subtasks with appropriate tools and models
4. Monitor agent progress and handle failures
5. Aggregate results and present final output

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

> Large Language Model (LLM) applications are vulnerable to prompt injection and context manipulation attacks that traditional security models cannot prevent. We introduce two novel primitives--authenticated prompts and authenticated context--that provide cryptographically verifiable provenance across LLM workflows. Authenticated prompts enable self-contained lineage verification, while authenticated context uses tamper-evident hash chains to ensure integrity of dynamic inputs. Building on these p

Refer to the [full paper](https://arxiv.org/abs/2602.10481v1) for detailed methodology.