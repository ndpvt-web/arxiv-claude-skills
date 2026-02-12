---
name: "patchguru-patch-oracle-inference"
description: "As software systems evolve, patches may unintentionally alter program behavior. Implements techniques from the paper 'PatchGuru: Patch Oracle Inference from Natural Language Artifacts with Large Language Models' for analyze code for bugs, vulnerabilities, and quality issues. Use when tasks involve (code analysis), (testing), (data processing), (search & retrieval) or when the user references techniques from this research area."
---

# PatchGuru: Patch Oracle Inference from Natural Language Artifacts with Large Language Models

**Source:** [https://arxiv.org/abs/2602.05270v1](https://arxiv.org/abs/2602.05270v1)
**Category:** cs.SE | **Published:** 2026-02-05 | **Skill Score:** 65
**Authors:** Thanh Le-Cong, Bach Le, Toby Murray...

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

## Testing Approach

- Generate unit tests covering happy path and edge cases
- Include boundary value tests
- Test error handling paths
- Aim for high code coverage

## Research Context

> As software systems evolve, patches may unintentionally alter program behavior. Validating patches against their intended semantics is difficult due to incomplete regression tests and informal, non-executable natural language (NL) descriptions of patch intent. We present PatchGuru, the first automated technique that infers executable patch specifications from real-world pull requests (PRs). Given a PR, PatchGuru uses large language models (LLMs) to extract developer intent from NL artifacts and 

Refer to the [full paper](https://arxiv.org/abs/2602.05270v1) for detailed methodology.