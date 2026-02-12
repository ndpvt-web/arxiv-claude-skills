---
name: "dont-believe-everything-you-read-understanding"
description: "The Model Context Protocol (MCP) enables large language models to invoke external tools through natural-language descriptions, forming the foundation of many AI agent applications. Implements techniques from the paper 'Don't believe everything you read: Understanding and Measuring MCP Behavior under Misleading Tool Descriptions' for analyze code for bugs, vulnerabilities, and quality issues. Use when tasks involve (code analysis), (data processing), (agent framework), (security) or when the user references techniques from this research area."
---

# Don't believe everything you read: Understanding and Measuring MCP Behavior under Misleading Tool Descriptions

**Source:** [https://arxiv.org/abs/2602.03580v1](https://arxiv.org/abs/2602.03580v1)
**Category:** cs.CR | **Published:** 2026-02-03 | **Skill Score:** 62
**Authors:** Zhihao Li, Boyang Ma, Xuelong Dai...

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

> The Model Context Protocol (MCP) enables large language models to invoke external tools through natural-language descriptions, forming the foundation of many AI agent applications. However, MCP does not enforce consistency between documented tool behavior and actual code execution, even though MCP Servers often run with broad system privileges. This gap introduces a largely unexplored security risk. We study how mismatches between externally presented tool descriptions and underlying implementat

Refer to the [full paper](https://arxiv.org/abs/2602.03580v1) for detailed methodology.