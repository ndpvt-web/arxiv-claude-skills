---
name: "beyond-functionlevel-analysis-contextaware"
description: "Recent progress in ML and LLMs has improved vulnerability detection, and recent datasets have reduced label noise and unrelated code changes. Implements techniques from the paper 'Beyond Function-Level Analysis: Context-Aware Reasoning for Inter-Procedural Vulnerability Detection' for analyze code for bugs, vulnerabilities, and quality issues. Use when tasks involve (code analysis), (agent framework), (security) or when the user references techniques from this research area."
---

# Beyond Function-Level Analysis: Context-Aware Reasoning for Inter-Procedural Vulnerability Detection

**Source:** [https://arxiv.org/abs/2602.06751v1](https://arxiv.org/abs/2602.06751v1)
**Category:** cs.CR | **Published:** 2026-02-06 | **Skill Score:** 67
**Authors:** Yikun Li, Ting Zhang, Jieke Shi...

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

> Recent progress in ML and LLMs has improved vulnerability detection, and recent datasets have reduced label noise and unrelated code changes. However, most existing approaches still operate at the function level, where models are asked to predict whether a single function is vulnerable without inter-procedural context. In practice, vulnerability presence and root cause often depend on contextual information. Naively appending such context is not a reliable solution: real-world context is long, r

Refer to the [full paper](https://arxiv.org/abs/2602.06751v1) for detailed methodology.