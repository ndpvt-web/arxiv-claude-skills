---
name: "scaling-testdriven-code-generation"
description: "Test-driven development (TDD) has been adopted to improve Large Language Model (LLM)-based code generation by using tests as executable specifications. Implements techniques from the paper 'Scaling Test-Driven Code Generation from Functions to Classes: An Empirical Study' for generate code from natural language descriptions. Use when tasks involve (code generation), (search & retrieval) or when the user references techniques from this research area."
---

# Scaling Test-Driven Code Generation from Functions to Classes: An Empirical Study

**Source:** [https://arxiv.org/abs/2602.03557v1](https://arxiv.org/abs/2602.03557v1)
**Category:** cs.SE | **Published:** 2026-02-03 | **Skill Score:** 66
**Authors:** Yunhao Liang, Ruixuan Ying, Shiwen Ni...

## Core Capability

Generate code from natural language descriptions.

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

> Test-driven development (TDD) has been adopted to improve Large Language Model (LLM)-based code generation by using tests as executable specifications. However, existing TDD-style code generation studies are largely limited to function-level tasks, leaving class-level synthesis where multiple methods interact through shared state and call dependencies underexplored. In this paper, we scale test-driven code generation from functions to classes via an iterative TDD framework. Our approach first an

Refer to the [full paper](https://arxiv.org/abs/2602.03557v1) for detailed methodology.