---
name: "can-llms-do-rocket"
description: "Large Language Models (LLMs) have demonstrated remarkable proficiency in code generation and general reasoning, yet their capacity for autonomous multi-stage planning in high-dimensional, physicall... Implements techniques from the paper 'Can LLMs Do Rocket Science? Exploring the Limits of Complex Reasoning with GTOC 12' for generate code from natural language descriptions. Use when tasks involve (code generation), (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# Can LLMs Do Rocket Science? Exploring the Limits of Complex Reasoning with GTOC 12

**Source:** [https://arxiv.org/abs/2602.03630v1](https://arxiv.org/abs/2602.03630v1)
**Category:** cs.AI | **Published:** 2026-02-03 | **Skill Score:** 66
**Authors:** Iñaki del Campo, Pablo Cuervo, Victor Rodriguez-Fernandez...

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

> Large Language Models (LLMs) have demonstrated remarkable proficiency in code generation and general reasoning, yet their capacity for autonomous multi-stage planning in high-dimensional, physically constrained environments remains an open research question. This study investigates the limits of current AI agents by evaluating them against the 12th Global Trajectory Optimization Competition (GTOC 12), a complex astrodynamics challenge requiring the design of a large-scale asteroid mining campaig

Refer to the [full paper](https://arxiv.org/abs/2602.03630v1) for detailed methodology.