---
name: "patch-to-poc-a-systematic-study"
description: "Autonomous large language model (LLM) based systems have recently shown promising results across a range of cybersecurity tasks. Implements techniques from the paper 'Patch-to-PoC: A Systematic Study of Agentic LLM Systems for Linux Kernel N-Day Reproduction' for analyze code for bugs, vulnerabilities, and quality issues. Use when tasks involve (code analysis), (agent framework), (security), (design & ui) or when the user references techniques from this research area."
---

# Patch-to-PoC: A Systematic Study of Agentic LLM Systems for Linux Kernel N-Day Reproduction

**Source:** [https://arxiv.org/abs/2602.07287v1](https://arxiv.org/abs/2602.07287v1)
**Category:** cs.CR | **Published:** 2026-02-07 | **Skill Score:** 74
**Authors:** Juefei Pu, Xingyu Li, Haonan Li...

## Core Capability

Analyze code for bugs, vulnerabilities, and quality issues.

## Key Techniques

- **Proposed technique:** the first large-sca

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

> Autonomous large language model (LLM) based systems have recently shown promising results across a range of cybersecurity tasks. However, there is no systematic study on their effectiveness in autonomously reproducing Linux kernel vulnerabilities with concrete proofs-of-concept (PoCs). Owing to the size, complexity, and low-level nature of the Linux kernel, such tasks are widely regarded as particularly challenging for current LLM-based approaches.   In this paper, we present the first large-sca

Refer to the [full paper](https://arxiv.org/abs/2602.07287v1) for detailed methodology.