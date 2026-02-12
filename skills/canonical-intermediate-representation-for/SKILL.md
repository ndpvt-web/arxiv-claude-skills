---
name: "canonical-intermediate-representation-for"
description: "Automatically formulating optimization models from natural language descriptions is a growing focus in operations research, yet current LLM-based approaches struggle with the composite constraints ... Implements techniques from the paper 'Canonical Intermediate Representation for LLM-based optimization problem formulation and code generation' for generate code from natural language descriptions. Use when tasks involve (code generation), (search & retrieval), (agent framework), (database & query) or when the user references techniques from this research area."
---

# Canonical Intermediate Representation for LLM-based optimization problem formulation and code generation

**Source:** [https://arxiv.org/abs/2602.02029v1](https://arxiv.org/abs/2602.02029v1)
**Category:** cs.AI | **Published:** 2026-02-02 | **Skill Score:** 83
**Authors:** Zhongyuan Lyu, Shuoyu Hu, Lujie Liu...

## Core Capability

Generate code from natural language descriptions.

## Key Techniques

- **Proposed technique:** the canonical intermediate representation (cir): a schema that llms explicitly generate between problem descriptions and optimization models

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

> Automatically formulating optimization models from natural language descriptions is a growing focus in operations research, yet current LLM-based approaches struggle with the composite constraints and appropriate modeling paradigms required by complex operational rules. To address this, we introduce the Canonical Intermediate Representation (CIR): a schema that LLMs explicitly generate between problem descriptions and optimization models. CIR encodes the semantics of operational rules through co

Refer to the [full paper](https://arxiv.org/abs/2602.02029v1) for detailed methodology.