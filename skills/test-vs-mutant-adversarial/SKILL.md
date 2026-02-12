---
name: "test-vs-mutant-adversarial"
description: "Software testing is a critical, yet resource-intensive phase of the software development lifecycle. Implements techniques from the paper 'Test vs Mutant: Adversarial LLM Agents for Robust Unit Test Generation' for analyze code for bugs, vulnerabilities, and quality issues. Use when tasks involve (code analysis), (testing), (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# Test vs Mutant: Adversarial LLM Agents for Robust Unit Test Generation

**Source:** [https://arxiv.org/abs/2602.08146v2](https://arxiv.org/abs/2602.08146v2)
**Category:** cs.SE | **Published:** 2026-02-08 | **Skill Score:** 69
**Authors:** Pengyu Chang, Yixiong Fang, Silin Chen...

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

## Agent Coordination

- Decompose complex tasks into independent subtasks where possible
- Use parallel execution for independent subtasks
- Implement error recovery and retry logic
- Aggregate partial results when full completion fails

## Research Context

> Software testing is a critical, yet resource-intensive phase of the software development lifecycle. Over the years, various automated tools have been developed to aid in this process. Search-based approaches typically achieve high coverage but produce tests with low readability, whereas large language model (LLM)-based methods generate more human-readable tests but often suffer from low coverage and compilability. While the majority of research efforts have focused on improving test coverage and

Refer to the [full paper](https://arxiv.org/abs/2602.08146v2) for detailed methodology.