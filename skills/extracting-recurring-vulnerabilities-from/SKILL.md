---
name: "extracting-recurring-vulnerabilities-from"
description: "LLMs are increasingly used for code generation, but their outputs often follow recurring templates that can induce predictable vulnerabilities. Implements techniques from the paper 'Extracting Recurring Vulnerabilities from Black-Box LLM-Generated Software' for generate code from natural language descriptions. Use when tasks involve (code generation), (code analysis), (search & retrieval), (security), (design & ui) or when the user references techniques from this research area."
---

# Extracting Recurring Vulnerabilities from Black-Box LLM-Generated Software

**Source:** [https://arxiv.org/abs/2602.04894v2](https://arxiv.org/abs/2602.04894v2)
**Category:** cs.CR | **Published:** 2026-02-02 | **Skill Score:** 68
**Authors:** Tomer Kordonsky, Maayan Yamin, Noam Benzimra...

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

## Security Considerations

- Check for OWASP Top 10 vulnerabilities
- Validate all user inputs at system boundaries
- Use parameterized queries for database access
- Follow the principle of least privilege

## Research Context

> LLMs are increasingly used for code generation, but their outputs often follow recurring templates that can induce predictable vulnerabilities. We study \emph{vulnerability persistence} in LLM-generated software and introduce \emph{Feature--Security Table (FSTab)} with two components. First, FSTab enables a black-box attack that predicts likely backend vulnerabilities from observable frontend features and knowledge of the source LLM, without access to backend code or source code. Second, FSTab p

Refer to the [full paper](https://arxiv.org/abs/2602.04894v2) for detailed methodology.