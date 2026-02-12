---
name: "qrs-a-rulesynthesizing-neurosymbolic"
description: "Static Application Security Testing (SAST) tools are integral to modern DevSecOps pipelines, yet tools like CodeQL, Semgrep, and SonarQube remain fundamentally constrained: they require expert-craf... Implements techniques from the paper 'QRS: A Rule-Synthesizing Neuro-Symbolic Triad for Autonomous Vulnerability Discovery' for analyze code for bugs, vulnerabilities, and quality issues. Use when tasks involve (code analysis), (data processing), (search & retrieval), (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# QRS: A Rule-Synthesizing Neuro-Symbolic Triad for Autonomous Vulnerability Discovery

**Source:** [https://arxiv.org/abs/2602.09774v1](https://arxiv.org/abs/2602.09774v1)
**Category:** cs.CR | **Published:** 2026-02-10 | **Skill Score:** 74
**Authors:** George Tsigkourakos, Constantinos Patsakis

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

## Agent Coordination

- Decompose complex tasks into independent subtasks where possible
- Use parallel execution for independent subtasks
- Implement error recovery and retry logic
- Aggregate partial results when full completion fails

## Research Context

> Static Application Security Testing (SAST) tools are integral to modern DevSecOps pipelines, yet tools like CodeQL, Semgrep, and SonarQube remain fundamentally constrained: they require expert-crafted queries, generate excessive false positives, and detect only predefined vulnerability patterns. Recent work has explored augmenting SAST with Large Language Models (LLMs), but these approaches typically use LLMs to triage existing tool outputs rather than to reason about vulnerability semantics dir

Refer to the [full paper](https://arxiv.org/abs/2602.09774v1) for detailed methodology.