---
name: "sifting-the-noise-a"
description: "Static Application Security Testing (SAST) tools are essential for identifying software vulnerabilities, but they often produce a high volume of false positives (FPs), imposing a substantial manual... Implements techniques from the paper 'Sifting the Noise: A Comparative Study of LLM Agents in Vulnerability False Positive Filtering' for analyze code for bugs, vulnerabilities, and quality issues. Use when tasks involve (code analysis), (devops automation), (agent framework), (prompt engineering), (security) or when the user references techniques from this research area."
---

# Sifting the Noise: A Comparative Study of LLM Agents in Vulnerability False Positive Filtering

**Source:** [https://arxiv.org/abs/2601.22952v1](https://arxiv.org/abs/2601.22952v1)
**Category:** cs.SE | **Published:** 2026-01-30 | **Skill Score:** 95
**Authors:** Yunpeng Xiong, Ting Zhang

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

> Static Application Security Testing (SAST) tools are essential for identifying software vulnerabilities, but they often produce a high volume of false positives (FPs), imposing a substantial manual triage burden on developers. Recent advances in Large Language Model (LLM) agents offer a promising direction by enabling iterative reasoning, tool use, and environment interaction to refine SAST alerts. However, the comparative effectiveness of different LLM-based agent architectures for FP filtering

Refer to the [full paper](https://arxiv.org/abs/2601.22952v1) for detailed methodology.