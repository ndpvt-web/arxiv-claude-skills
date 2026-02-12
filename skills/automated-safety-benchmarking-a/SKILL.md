---
name: "automated-safety-benchmarking-a"
description: "Large vision-language models (LVLMs) exhibit remarkable capabilities in cross-modal tasks but face significant safety challenges, which undermine their reliability in real-world applications. Implements techniques from the paper 'Automated Safety Benchmarking: A Multi-agent Pipeline for LVLMs' for analyze code for bugs, vulnerabilities, and quality issues. Use when tasks involve (code analysis), (agent framework), (security) or when the user references techniques from this research area."
---

# Automated Safety Benchmarking: A Multi-agent Pipeline for LVLMs

**Source:** [https://arxiv.org/abs/2601.19507v1](https://arxiv.org/abs/2601.19507v1)
**Category:** cs.CL | **Published:** 2026-01-27 | **Skill Score:** 60
**Authors:** Xiangyang Zhu, Yuan Tian, Zicheng Zhang...

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

> Large vision-language models (LVLMs) exhibit remarkable capabilities in cross-modal tasks but face significant safety challenges, which undermine their reliability in real-world applications. Efforts have been made to build LVLM safety evaluation benchmarks to uncover their vulnerability. However, existing benchmarks are hindered by their labor-intensive construction process, static complexity, and limited discriminative power. Thus, they may fail to keep pace with rapidly evolving models and em

Refer to the [full paper](https://arxiv.org/abs/2601.19507v1) for detailed methodology.