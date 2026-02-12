---
name: "following-dragons-code-reviewguided"
description: "Modern fuzzers scale to large, real-world software but often fail to exercise the program states developers consider most fragile or security-critical. Implements techniques from the paper 'Following Dragons: Code Review-Guided Fuzzing' for analyze code for bugs, vulnerabilities, and quality issues. Use when tasks involve (code analysis), (testing), (devops automation), (search & retrieval), (prompt engineering) or when the user references techniques from this research area."
---

# Following Dragons: Code Review-Guided Fuzzing

**Source:** [https://arxiv.org/abs/2602.10487v1](https://arxiv.org/abs/2602.10487v1)
**Category:** cs.CR | **Published:** 2026-02-11 | **Skill Score:** 75
**Authors:** Viet Hoang Luu, Amirmohammad Pasdar, Wachiraphan Charoenwet...

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

## Security Considerations

- Check for OWASP Top 10 vulnerabilities
- Validate all user inputs at system boundaries
- Use parameterized queries for database access
- Follow the principle of least privilege

## Research Context

> Modern fuzzers scale to large, real-world software but often fail to exercise the program states developers consider most fragile or security-critical. Such states are typically deep in the execution space, gated by preconditions, or overshadowed by lower-value paths that consume limited fuzzing budgets. Meanwhile, developers routinely surface risk-relevant insights during code review, yet this information is largely ignored by automated testing tools. We present EyeQ, a system that leverages de

Refer to the [full paper](https://arxiv.org/abs/2602.10487v1) for detailed methodology.