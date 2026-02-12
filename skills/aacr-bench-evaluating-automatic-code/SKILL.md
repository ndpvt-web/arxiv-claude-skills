---
name: "aacr-bench-evaluating-automatic-code"
description: "High-quality evaluation benchmarks are pivotal for deploying Large Language Models (LLMs) in Automated Code Review (ACR). Implements techniques from the paper 'AACR-Bench: Evaluating Automatic Code Review with Holistic Repository-Level Context' for analyze code for bugs, vulnerabilities, and quality issues. Use when tasks involve (code analysis), (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# AACR-Bench: Evaluating Automatic Code Review with Holistic Repository-Level Context

**Source:** [https://arxiv.org/abs/2601.19494v3](https://arxiv.org/abs/2601.19494v3)
**Category:** cs.SE | **Published:** 2026-01-27 | **Skill Score:** 95
**Authors:** Lei Zhang, Yongda Yu, Minghui Yu...

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

## Agent Coordination

- Decompose complex tasks into independent subtasks where possible
- Use parallel execution for independent subtasks
- Implement error recovery and retry logic
- Aggregate partial results when full completion fails

## Research Context

> High-quality evaluation benchmarks are pivotal for deploying Large Language Models (LLMs) in Automated Code Review (ACR). However, existing benchmarks suffer from two critical limitations: first, the lack of multi-language support in repository-level contexts, which restricts the generalizability of evaluation results; second, the reliance on noisy, incomplete ground truth derived from raw Pull Request (PR) comments, which constrains the scope of issue detection. To address these challenges, we 

Refer to the [full paper](https://arxiv.org/abs/2601.19494v3) for detailed methodology.