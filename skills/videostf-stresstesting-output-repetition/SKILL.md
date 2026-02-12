---
name: "videostf-stresstesting-output-repetition"
description: "Video Large Language Models (VideoLLMs) have recently achieved strong performance in video understanding tasks. Implements techniques from the paper 'VideoSTF: Stress-Testing Output Repetition in Video Large Language Models' for analyze code for bugs, vulnerabilities, and quality issues. Use when tasks involve (code analysis), (security) or when the user references techniques from this research area."
---

# VideoSTF: Stress-Testing Output Repetition in Video Large Language Models

**Source:** [https://arxiv.org/abs/2602.10639v1](https://arxiv.org/abs/2602.10639v1)
**Category:** cs.CV | **Published:** 2026-02-11 | **Skill Score:** 77
**Authors:** Yuxin Cao, Wei Song, Shangzhi Xu...

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

## Research Context

> Video Large Language Models (VideoLLMs) have recently achieved strong performance in video understanding tasks. However, we identify a previously underexplored generation failure: severe output repetition, where models degenerate into self-reinforcing loops of repeated phrases or sentences. This failure mode is not captured by existing VideoLLM benchmarks, which focus primarily on task accuracy and factual correctness. We introduce VideoSTF, the first framework for systematically measuring and s

Refer to the [full paper](https://arxiv.org/abs/2602.10639v1) for detailed methodology.