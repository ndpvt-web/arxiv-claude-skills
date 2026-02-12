---
name: "projdevbench-benchmarking-ai-coding"
description: "Recent coding agents can generate complete codebases from simple prompts, yet existing evaluations focus on issue-level bug fixing and lag behind end-to-end development. Implements techniques from the paper 'ProjDevBench: Benchmarking AI Coding Agents on End-to-End Project Development' for analyze code for bugs, vulnerabilities, and quality issues. Use when tasks involve (code analysis), (code transformation), (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# ProjDevBench: Benchmarking AI Coding Agents on End-to-End Project Development

**Source:** [https://arxiv.org/abs/2602.01655v2](https://arxiv.org/abs/2602.01655v2)
**Category:** cs.AI | **Published:** 2026-02-02 | **Skill Score:** 88
**Authors:** Pengrui Lu, Shiqi Zhang, Yunzhong Hou...

## Core Capability

Analyze code for bugs, vulnerabilities, and quality issues.

## Key Techniques

- **Proposed technique:** projdevbench

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

> Recent coding agents can generate complete codebases from simple prompts, yet existing evaluations focus on issue-level bug fixing and lag behind end-to-end development. We introduce ProjDevBench, an end-to-end benchmark that provides project requirements to coding agents and evaluates the resulting repositories. Combining Online Judge (OJ) testing with LLM-assisted code review, the benchmark evaluates agents on (1) system architecture design, (2) functional correctness, and (3) iterative soluti

Refer to the [full paper](https://arxiv.org/abs/2602.01655v2) for detailed methodology.