---
name: "swe-manager-selecting-and-synthesizing"
description: "Large language model (LLM) research in software engineering has largely focused on tasks such as code generation and bug repair. Implements techniques from the paper 'SWE-Manager: Selecting and Synthesizing Golden Proposals Before Coding' for generate code from natural language descriptions. Use when tasks involve (code generation), (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# SWE-Manager: Selecting and Synthesizing Golden Proposals Before Coding

**Source:** [https://arxiv.org/abs/2601.22956v1](https://arxiv.org/abs/2601.22956v1)
**Category:** cs.SE | **Published:** 2026-01-30 | **Skill Score:** 67
**Authors:** Boyin Tan, Haoning Deng, Junyuan Zhang...

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

> Large language model (LLM) research in software engineering has largely focused on tasks such as code generation and bug repair. In practice, teams often draft multiple candidate proposals for fixing an issue and then deliberate on one golden proposal for implementation. This selection requires not only assessing the issue's scope, impact, and urgency, but also a clear understanding of each proposal's strengths and weaknesses. A good selection could make issue resolution more reliable while redu

Refer to the [full paper](https://arxiv.org/abs/2601.22956v1) for detailed methodology.