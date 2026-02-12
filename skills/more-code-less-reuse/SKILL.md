---
name: "more-code-less-reuse"
description: "Large Language Model (LLM) Agents are advancing quickly, with the increasing leveraging of LLM Agents to assist in development tasks such as code generation. Implements techniques from the paper 'More Code, Less Reuse: Investigating Code Quality and Reviewer Sentiment towards AI-generated Pull Requests' for generate code from natural language descriptions. Use when tasks involve (code generation), (code analysis), (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# More Code, Less Reuse: Investigating Code Quality and Reviewer Sentiment towards AI-generated Pull Requests

**Source:** [https://arxiv.org/abs/2601.21276v1](https://arxiv.org/abs/2601.21276v1)
**Category:** cs.SE | **Published:** 2026-01-29 | **Skill Score:** 81
**Authors:** Haoming Huang, Pongchai Jaisri, Shota Shimizu...

## Core Capability

Generate code from natural language descriptions.

## Key Techniques

- **Leverages:** of llm agents to assist in development tasks such as code generation

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

> Large Language Model (LLM) Agents are advancing quickly, with the increasing leveraging of LLM Agents to assist in development tasks such as code generation. While LLM Agents accelerate code generation, studies indicate they may introduce adverse effects on development. However, existing metrics solely measure pass rates, failing to reflect impacts on long-term maintainability and readability, and failing to capture human intuitive evaluations of PR. To increase the comprehensiveness of this pro

Refer to the [full paper](https://arxiv.org/abs/2601.21276v1) for detailed methodology.