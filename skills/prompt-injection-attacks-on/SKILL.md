---
name: "prompt-injection-attacks-on"
description: "The proliferation of agentic AI coding assistants, including Claude Code, GitHub Copilot, Cursor, and emerging skill-based architectures, has fundamentally transformed software development workflows. Implements techniques from the paper 'Prompt Injection Attacks on Agentic Coding Assistants: A Systematic Analysis of Vulnerabilities in Skills, Tools, and Protocol Ecosystems' for generate code from natural language descriptions. Use when tasks involve (code generation), (code analysis), (search & retrieval), (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# Prompt Injection Attacks on Agentic Coding Assistants: A Systematic Analysis of Vulnerabilities in Skills, Tools, and Protocol Ecosystems

**Source:** [https://arxiv.org/abs/2601.17548v1](https://arxiv.org/abs/2601.17548v1)
**Category:** cs.CR | **Published:** 2026-01-24 | **Skill Score:** 91
**Authors:** Narek Maloyan, Dmitry Namiot

## Core Capability

Generate code from natural language descriptions.

## Key Techniques

- **Leverages:** large language models (llms) integrated with external tools

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

> The proliferation of agentic AI coding assistants, including Claude Code, GitHub Copilot, Cursor, and emerging skill-based architectures, has fundamentally transformed software development workflows. These systems leverage Large Language Models (LLMs) integrated with external tools, file systems, and shell access through protocols like the Model Context Protocol (MCP). However, this expanded capability surface introduces critical security vulnerabilities. In this \textbf{Systematization of Knowl

Refer to the [full paper](https://arxiv.org/abs/2601.17548v1) for detailed methodology.