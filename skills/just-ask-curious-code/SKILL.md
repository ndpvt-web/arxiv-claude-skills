---
name: "just-ask-curious-code"
description: "Autonomous code agents built on large language models are reshaping software and AI development through tool use, long-horizon reasoning, and self-directed interaction. Implements techniques from the paper 'Just Ask: Curious Code Agents Reveal System Prompts in Frontier LLMs' for analyze code for bugs, vulnerabilities, and quality issues. Use when tasks involve (code analysis), (agent framework), (prompt engineering), (security) or when the user references techniques from this research area."
---

# Just Ask: Curious Code Agents Reveal System Prompts in Frontier LLMs

**Source:** [https://arxiv.org/abs/2601.21233v1](https://arxiv.org/abs/2601.21233v1)
**Category:** cs.AI | **Published:** 2026-01-29 | **Skill Score:** 80
**Authors:** Xiang Zheng, Yutao Wu, Hanxun Huang...

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

> Autonomous code agents built on large language models are reshaping software and AI development through tool use, long-horizon reasoning, and self-directed interaction. However, this autonomy introduces a previously unrecognized security risk: agentic interaction fundamentally expands the LLM attack surface, enabling systematic probing and recovery of hidden system prompts that guide model behavior. We identify system prompt extraction as an emergent vulnerability intrinsic to code agents and pr

Refer to the [full paper](https://arxiv.org/abs/2601.21233v1) for detailed methodology.