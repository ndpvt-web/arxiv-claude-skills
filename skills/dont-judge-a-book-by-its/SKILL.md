---
name: "dont-judge-a-book-by-its"
description: "Tasks such as solving arithmetic equations, evaluating truth tables, and completing syllogisms are handled well by large language models (LLMs) in their standard form, but they often fail when the ... Implements techniques from the paper 'Don't Judge a Book by its Cover: Testing LLMs' Robustness Under Logical Obfuscation' for analyze code for bugs, vulnerabilities, and quality issues. Use when tasks involve (code analysis), (search & retrieval), (agent framework), (prompt engineering), (security) or when the user references techniques from this research area."
---

# Don't Judge a Book by its Cover: Testing LLMs' Robustness Under Logical Obfuscation

**Source:** [https://arxiv.org/abs/2602.01132v1](https://arxiv.org/abs/2602.01132v1)
**Category:** cs.CL | **Published:** 2026-02-01 | **Skill Score:** 65
**Authors:** Abhilekh Borah, Shubhra Ghosh, Kedar Joshi...

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

> Tasks such as solving arithmetic equations, evaluating truth tables, and completing syllogisms are handled well by large language models (LLMs) in their standard form, but they often fail when the same problems are posed in logically equivalent yet obfuscated formats. To study this vulnerability, we introduce Logifus, a structure-preserving logical obfuscation framework, and, utilizing this, we present LogiQAte, a first-of-its-kind diagnostic benchmark with 1,108 questions across four reasoning 

Refer to the [full paper](https://arxiv.org/abs/2602.01132v1) for detailed methodology.