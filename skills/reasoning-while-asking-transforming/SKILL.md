---
name: "reasoning-while-asking-transforming"
description: "Reasoning-oriented Large Language Models (LLMs) have achieved remarkable progress with Chain-of-Thought (CoT) prompting, yet they remain fundamentally limited by a \emph{blind self-thinking} paradi... Implements techniques from the paper 'Reasoning While Asking: Transforming Reasoning Large Language Models from Passive Solvers to Proactive Inquirers' for generate code from natural language descriptions. Use when tasks involve (code generation), (data processing), (search & retrieval), (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# Reasoning While Asking: Transforming Reasoning Large Language Models from Passive Solvers to Proactive Inquirers

**Source:** [https://arxiv.org/abs/2601.22139v1](https://arxiv.org/abs/2601.22139v1)
**Category:** cs.CL | **Published:** 2026-01-29 | **Skill Score:** 100
**Authors:** Xin Chen, Feng Jiang, Yiqian Zhang...

## Core Capability

Generate code from natural language descriptions.

## Key Techniques

- **Proposed technique:** proactive interactive reasoning (pir)
- **Chain-of-thought reasoning** for improved step-by-step problem solving

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

## Agent Coordination

- Decompose complex tasks into independent subtasks where possible
- Use parallel execution for independent subtasks
- Implement error recovery and retry logic
- Aggregate partial results when full completion fails

## Research Context

> Reasoning-oriented Large Language Models (LLMs) have achieved remarkable progress with Chain-of-Thought (CoT) prompting, yet they remain fundamentally limited by a \emph{blind self-thinking} paradigm: performing extensive internal reasoning even when critical information is missing or ambiguous. We propose Proactive Interactive Reasoning (PIR), a new reasoning paradigm that transforms LLMs from passive solvers into proactive inquirers that interleave reasoning with clarification. Unlike existing

Refer to the [full paper](https://arxiv.org/abs/2601.22139v1) for detailed methodology.