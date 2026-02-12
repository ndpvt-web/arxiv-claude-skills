---
name: "how-do-agents-refactor"
description: "Software development agents such as Claude Code, GitHub Copilot, Cursor Agent, Devin, and OpenAI Codex are being increasingly integrated into developer workflows. Implements techniques from the paper 'How do Agents Refactor: An Empirical Study' for generate code from natural language descriptions. Use when tasks involve (code generation), (code analysis), (code transformation), (agent framework) or when the user references techniques from this research area."
---

# How do Agents Refactor: An Empirical Study

**Source:** [https://arxiv.org/abs/2601.20160v1](https://arxiv.org/abs/2601.20160v1)
**Category:** cs.SE | **Published:** 2026-01-28 | **Skill Score:** 58
**Authors:** Lukas Ottenhof, Daniel Penner, Abram Hindle...

## Core Capability

Generate code from natural language descriptions.

## Key Techniques

- **Proposed technique:** the first analysis of agentic refactoring pull requests in java

## Workflow

1. Parse the user's natural language description of desired functionality
2. Identify the target programming language and framework
3. Generate well-structured, idiomatic code following best practices
4. Include appropriate error handling, types, and documentation
5. Validate generated code for correctness and security

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

> Software development agents such as Claude Code, GitHub Copilot, Cursor Agent, Devin, and OpenAI Codex are being increasingly integrated into developer workflows. While prior work has evaluated agent capabilities for code completion and task automation, there is little work investigating how these agents perform Java refactoring in practice, the types of changes they make, and their impact on code quality. In this study, we present the first analysis of agentic refactoring pull requests in Java,

Refer to the [full paper](https://arxiv.org/abs/2601.20160v1) for detailed methodology.