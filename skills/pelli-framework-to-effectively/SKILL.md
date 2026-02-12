---
name: "pelli-framework-to-effectively"
description: "Recent studies have revealed that when LLMs are appropriately prompted and configured, they demonstrate mixed results. Implements techniques from the paper 'PELLI: Framework to effectively integrate LLMs for quality software generation' for analyze code for bugs, vulnerabilities, and quality issues. Use when tasks involve (code analysis), (search & retrieval), (prompt engineering) or when the user references techniques from this research area."
---

# PELLI: Framework to effectively integrate LLMs for quality software generation

**Source:** [https://arxiv.org/abs/2602.10808v1](https://arxiv.org/abs/2602.10808v1)
**Category:** cs.SE | **Published:** 2026-02-11 | **Skill Score:** 61
**Authors:** Rasmus Krebs, Somnath Mazumdar

## Core Capability

Analyze code for bugs, vulnerabilities, and quality issues.

## Key Techniques

- **Proposed technique:** a comprehensive code quality assessment framework called programmatic excellence via llm iteration (pelli)
- **Achievement:** the baseline performance

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

> Recent studies have revealed that when LLMs are appropriately prompted and configured, they demonstrate mixed results. Such results often meet or exceed the baseline performance. However, these comparisons have two primary issues. First, they mostly considered only reliability as a comparison metric and selected a few LLMs (such as Codex and ChatGPT) for comparision. This paper proposes a comprehensive code quality assessment framework called Programmatic Excellence via LLM Iteration (PELLI). PE

Refer to the [full paper](https://arxiv.org/abs/2602.10808v1) for detailed methodology.