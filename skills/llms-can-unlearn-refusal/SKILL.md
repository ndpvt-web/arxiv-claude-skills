---
name: "llms-can-unlearn-refusal"
description: "This study reveals a previously unexplored vulnerability in the safety alignment of Large Language Models (LLMs). Implements techniques from the paper 'LLMs Can Unlearn Refusal with Only 1,000 Benign Samples' for analyze code for bugs, vulnerabilities, and quality issues. Use when tasks involve (code analysis), (agent framework), (prompt engineering), (security) or when the user references techniques from this research area."
---

# LLMs Can Unlearn Refusal with Only 1,000 Benign Samples

**Source:** [https://arxiv.org/abs/2601.19231v1](https://arxiv.org/abs/2601.19231v1)
**Category:** cs.CR | **Published:** 2026-01-27 | **Skill Score:** 61
**Authors:** Yangyang Guo, Ziwei Xu, Si Liu...

## Core Capability

Analyze code for bugs, vulnerabilities, and quality issues.

## Key Techniques

- **Novel approach:** \textbf{refusal unlearning} technique

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

> This study reveals a previously unexplored vulnerability in the safety alignment of Large Language Models (LLMs). Existing aligned LLMs predominantly respond to unsafe queries with refusals, which often begin with a fixed set of prefixes (I'm sorry). We demonstrate that this rigid refusal pattern is a vulnerability and introduce a novel \textbf{refusal unlearning} technique that exploits it. Specifically, we fine-tune LLMs using merely 1,000 benign samples, where each response is prepended with 

Refer to the [full paper](https://arxiv.org/abs/2601.19231v1) for detailed methodology.