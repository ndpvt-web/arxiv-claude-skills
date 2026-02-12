---
name: "chart-specification-structural-representations"
description: "Vision-Language Models (VLMs) have shown promise in generating plotting code from chart images, yet achieving structural fidelity remains challenging. Implements techniques from the paper 'Chart Specification: Structural Representations for Incentivizing VLM Reasoning in Chart-to-Code Generation' for generate code from natural language descriptions. Use when tasks involve (code generation), (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# Chart Specification: Structural Representations for Incentivizing VLM Reasoning in Chart-to-Code Generation

**Source:** [https://arxiv.org/abs/2602.10880v1](https://arxiv.org/abs/2602.10880v1)
**Category:** cs.CV | **Published:** 2026-02-11 | **Skill Score:** 69
**Authors:** Minggui He, Mingchen Dai, Jian Zhang...

## Core Capability

Generate code from natural language descriptions.

## Key Techniques

- **Proposed technique:** chart specification
- **Achievement:** structural fidelity remains challenging

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

> Vision-Language Models (VLMs) have shown promise in generating plotting code from chart images, yet achieving structural fidelity remains challenging. Existing approaches largely rely on supervised fine-tuning, encouraging surface-level token imitation rather than faithful modeling of underlying chart structure, which often leads to hallucinated or semantically inconsistent outputs. We propose Chart Specification, a structured intermediate representation that shifts training from text imitation 

Refer to the [full paper](https://arxiv.org/abs/2602.10880v1) for detailed methodology.