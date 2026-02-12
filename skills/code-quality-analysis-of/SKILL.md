---
name: "code-quality-analysis-of"
description: "C/C++ is a prevalent programming language. Implements techniques from the paper 'Code Quality Analysis of Translations from C to Rust' for analyze code for bugs, vulnerabilities, and quality issues. Use when tasks involve (code analysis), (code transformation) or when the user references techniques from this research area."
---

# Code Quality Analysis of Translations from C to Rust

**Source:** [https://arxiv.org/abs/2602.00840v1](https://arxiv.org/abs/2602.00840v1)
**Category:** cs.SE | **Published:** 2026-01-31 | **Skill Score:** 65
**Authors:** Biruk Tadesse, Vikram Nitin, Mazin Salah...

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

> C/C++ is a prevalent programming language. Yet, it suffers from significant memory and thread-safety issues. Recent studies have explored automated translation of C/C++ to safer languages, such as Rust. However, these studies focused mostly on the correctness and safety of the translated code, which are indeed critical, but they left other important quality concerns (e.g., performance, robustness, and maintainability) largely unexplored. This work investigates strengths and weaknesses of three C

Refer to the [full paper](https://arxiv.org/abs/2602.00840v1) for detailed methodology.