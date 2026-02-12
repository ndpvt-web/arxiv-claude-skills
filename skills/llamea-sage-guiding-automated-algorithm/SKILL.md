---
name: "llamea-sage-guiding-automated-algorithm"
description: "Large language models have enabled automated algorithm design (AAD) by generating optimization algorithms directly from natural-language prompts. Implements techniques from the paper 'LLaMEA-SAGE: Guiding Automated Algorithm Design with Structural Feedback from Explainable AI' for generate code from natural language descriptions. Use when tasks involve (code generation), (search & retrieval), (prompt engineering) or when the user references techniques from this research area."
---

# LLaMEA-SAGE: Guiding Automated Algorithm Design with Structural Feedback from Explainable AI

**Source:** [https://arxiv.org/abs/2601.21511v1](https://arxiv.org/abs/2601.21511v1)
**Category:** cs.AI | **Published:** 2026-01-29 | **Skill Score:** 75
**Authors:** Niki van Stein, Anna V. Kononova, Lars Kotthoff...

## Core Capability

Generate code from natural language descriptions.

## Key Techniques

- **Proposed technique:** a mechanism for guiding aad using feedback constructed from graph-theoretic and complexity

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

## Research Context

> Large language models have enabled automated algorithm design (AAD) by generating optimization algorithms directly from natural-language prompts. While evolutionary frameworks such as LLaMEA demonstrate strong exploratory capabilities across the algorithm design space, their search dynamics are entirely driven by fitness feedback, leaving substantial information about the generated code unused. We propose a mechanism for guiding AAD using feedback constructed from graph-theoretic and complexity 

Refer to the [full paper](https://arxiv.org/abs/2601.21511v1) for detailed methodology.