---
name: "llms-security-trouble"
description: "We argue that when it comes to producing secure code with AI, the prevailing \"fighting fire with fire\" approach -- using probabilistic AI-based checkers or attackers to secure probabilistically gen... Implements techniques from the paper 'LLMs + Security = Trouble' for generate code from natural language descriptions. Use when tasks involve (code generation), (security) or when the user references techniques from this research area."
---

# LLMs + Security = Trouble

**Source:** [https://arxiv.org/abs/2602.08422v1](https://arxiv.org/abs/2602.08422v1)
**Category:** cs.CR | **Published:** 2026-02-09 | **Skill Score:** 72
**Authors:** Benjamin Livshits

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

> We argue that when it comes to producing secure code with AI, the prevailing "fighting fire with fire" approach -- using probabilistic AI-based checkers or attackers to secure probabilistically generated code -- fails to address the long tail of security bugs. As a result, systems may remain exposed to zero-day vulnerabilities that can be discovered by better-resourced or more persistent adversaries.   While neurosymbolic approaches that combine LLMs with formal methods are attractive in princip

Refer to the [full paper](https://arxiv.org/abs/2602.08422v1) for detailed methodology.