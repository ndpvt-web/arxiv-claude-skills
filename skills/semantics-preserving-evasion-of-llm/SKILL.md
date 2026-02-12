---
name: "semantics-preserving-evasion-of-llm"
description: "LLM-based vulnerability detectors are increasingly deployed in security-critical code review, yet their resilience to evasion under behavior-preserving edits remains poorly understood. Implements techniques from the paper 'Semantics-Preserving Evasion of LLM Vulnerability Detectors' for analyze code for bugs, vulnerabilities, and quality issues. Use when tasks involve (code analysis), (security) or when the user references techniques from this research area."
---

# Semantics-Preserving Evasion of LLM Vulnerability Detectors

**Source:** [https://arxiv.org/abs/2602.00305v1](https://arxiv.org/abs/2602.00305v1)
**Category:** cs.CR | **Published:** 2026-01-30 | **Skill Score:** 61
**Authors:** Luze Sun, Alina Oprea, Eric Wong

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

> LLM-based vulnerability detectors are increasingly deployed in security-critical code review, yet their resilience to evasion under behavior-preserving edits remains poorly understood. We evaluate detection-time integrity under a semantics-preserving threat model by instantiating diverse behavior-preserving code transformations on a unified C/C++ benchmark (N=5000), and introduce a metric of joint robustness across different attack methods/carriers. Across models, we observe a systemic failure o

Refer to the [full paper](https://arxiv.org/abs/2602.00305v1) for detailed methodology.