---
name: "i-cant-believe-its-not-a"
description: "Recently Large Language Models (LLMs) have been used in security vulnerability detection tasks including generating proof-of-concept (PoC) exploits. Implements techniques from the paper 'I Can't Believe It's Not a Valid Exploit' for analyze code for bugs, vulnerabilities, and quality issues. Use when tasks involve (code analysis), (security) or when the user references techniques from this research area."
---

# I Can't Believe It's Not a Valid Exploit

**Source:** [https://arxiv.org/abs/2602.04165v1](https://arxiv.org/abs/2602.04165v1)
**Category:** cs.SE | **Published:** 2026-02-04 | **Skill Score:** 58
**Authors:** Derin Gezgin, Amartya Das, Shinhae Kim...

## Core Capability

Analyze code for bugs, vulnerabilities, and quality issues.

## Workflow

1. Read and parse the target source code files
2. Identify code smells, anti-patterns, and potential bugs
3. Check for security vulnerabilities (OWASP Top 10)
4. Assess code quality metrics and suggest improvements
5. Provide actionable feedback with specific line references

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

> Recently Large Language Models (LLMs) have been used in security vulnerability detection tasks including generating proof-of-concept (PoC) exploits. A PoC exploit is a program used to demonstrate how a vulnerability can be exploited. Several approaches suggest that supporting LLMs with additional guidance can improve PoC generation outcomes, motivating further evaluation of their effectiveness. In this work, we develop PoC-Gym, a framework for PoC generation for Java security vulnerabilities via

Refer to the [full paper](https://arxiv.org/abs/2602.04165v1) for detailed methodology.