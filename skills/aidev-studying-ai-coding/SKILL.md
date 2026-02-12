---
name: "aidev-studying-ai-coding"
description: "AI coding agents are rapidly transforming software engineering by performing tasks such as feature development, debugging, and testing. Implements techniques from the paper 'AIDev: Studying AI Coding Agents on GitHub' for generate code from natural language descriptions. Use when tasks involve (code generation), (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# AIDev: Studying AI Coding Agents on GitHub

**Source:** [https://arxiv.org/abs/2602.09185v1](https://arxiv.org/abs/2602.09185v1)
**Category:** cs.SE | **Published:** 2026-02-09 | **Skill Score:** 71
**Authors:** Hao Li, Haoxiang Zhang, Ahmed E. Hassan

## Core Capability

Generate code from natural language descriptions.

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

> AI coding agents are rapidly transforming software engineering by performing tasks such as feature development, debugging, and testing. Despite their growing impact, the research community lacks a comprehensive dataset capturing how these agents are used in real-world projects. To address this gap, we introduce AIDev, a large-scale dataset focused on agent-authored pull requests (Agentic-PRs) in real-world GitHub repositories. AIDev aggregates 932,791 Agentic-PRs produced by five agents: OpenAI 

Refer to the [full paper](https://arxiv.org/abs/2602.09185v1) for detailed methodology.