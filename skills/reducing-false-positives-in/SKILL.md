---
name: "reducing-false-positives-in"
description: "Static analysis tools (SATs) are widely adopted in both academia and industry for improving software quality, yet their practical use is often hindered by high false positive rates, especially in l... Implements techniques from the paper 'Reducing False Positives in Static Bug Detection with LLMs: An Empirical Study in Industry' for analyze code for bugs, vulnerabilities, and quality issues. Use when tasks involve (code analysis) or when the user references techniques from this research area."
---

# Reducing False Positives in Static Bug Detection with LLMs: An Empirical Study in Industry

**Source:** [https://arxiv.org/abs/2601.18844v1](https://arxiv.org/abs/2601.18844v1)
**Category:** cs.SE | **Published:** 2026-01-26 | **Skill Score:** 73
**Authors:** Xueying Du, Jiayi Feng, Yi Zou...

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

## Research Context

> Static analysis tools (SATs) are widely adopted in both academia and industry for improving software quality, yet their practical use is often hindered by high false positive rates, especially in large-scale enterprise systems. These false alarms demand substantial manual inspection, creating severe inefficiencies in industrial code review. While recent work has demonstrated the potential of large language models (LLMs) for false alarm reduction on open-source benchmarks, their effectiveness in 

Refer to the [full paper](https://arxiv.org/abs/2601.18844v1) for detailed methodology.