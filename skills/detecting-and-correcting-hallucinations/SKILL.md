---
name: "detecting-and-correcting-hallucinations"
description: "Large Language Models (LLMs) for code generation boost productivity but frequently introduce Knowledge Conflicting Hallucinations (KCHs), subtle, semantic errors, such as non-existent API parameter... Implements techniques from the paper 'Detecting and Correcting Hallucinations in LLM-Generated Code via Deterministic AST Analysis' for generate code from natural language descriptions. Use when tasks involve (code generation), (search & retrieval) or when the user references techniques from this research area."
---

# Detecting and Correcting Hallucinations in LLM-Generated Code via Deterministic AST Analysis

**Source:** [https://arxiv.org/abs/2601.19106v1](https://arxiv.org/abs/2601.19106v1)
**Category:** cs.SE | **Published:** 2026-01-27 | **Skill Score:** 75
**Authors:** Dipin Khati, Daniel Rodriguez-Cardenas, Paul Pantzer...

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

> Large Language Models (LLMs) for code generation boost productivity but frequently introduce Knowledge Conflicting Hallucinations (KCHs), subtle, semantic errors, such as non-existent API parameters, that evade linters and cause runtime failures. Existing mitigations like constrained decoding or non-deterministic LLM-in-the-loop repair are often unreliable for these errors. This paper investigates whether a deterministic, static-analysis framework can reliably detect \textit{and} auto-correct KC

Refer to the [full paper](https://arxiv.org/abs/2601.19106v1) for detailed methodology.