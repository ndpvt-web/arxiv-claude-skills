---
name: "clarify-before-you-draw"
description: "Large language models have recently enabled text-to-CAD systems that synthesize parametric CAD programs (e.g., CadQuery) from natural language prompts. Implements techniques from the paper 'Clarify Before You Draw: Proactive Agents for Robust Text-to-CAD Generation' for generate code from natural language descriptions. Use when tasks involve (code generation), (agent framework), (prompt engineering), (database & query) or when the user references techniques from this research area."
---

# Clarify Before You Draw: Proactive Agents for Robust Text-to-CAD Generation

**Source:** [https://arxiv.org/abs/2602.03045v1](https://arxiv.org/abs/2602.03045v1)
**Category:** cs.LG | **Published:** 2026-02-03 | **Skill Score:** 61
**Authors:** Bo Yuan, Zelin Zhao, Petr Molodyk...

## Core Capability

Generate code from natural language descriptions.

## Key Techniques

- **Proposed technique:** a proactive agentic framework for

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

> Large language models have recently enabled text-to-CAD systems that synthesize parametric CAD programs (e.g., CadQuery) from natural language prompts. In practice, however, geometric descriptions can be under-specified or internally inconsistent: critical dimensions may be missing and constraints may conflict. Existing fine-tuned models tend to reactively follow user instructions and hallucinate dimensions when the text is ambiguous. To address this, we propose a proactive agentic framework for

Refer to the [full paper](https://arxiv.org/abs/2602.03045v1) for detailed methodology.