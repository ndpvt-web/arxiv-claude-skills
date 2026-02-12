---
name: "swe-refactor-a-repositorylevel-benchmark"
description: "Large Language Models (LLMs) have recently attracted wide interest for tackling software engineering tasks. Implements techniques from the paper 'SWE-Refactor: A Repository-Level Benchmark for Real-World LLM-Based Code Refactoring' for generate code from natural language descriptions. Use when tasks involve (code generation), (code transformation), (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# SWE-Refactor: A Repository-Level Benchmark for Real-World LLM-Based Code Refactoring

**Source:** [https://arxiv.org/abs/2602.03712v1](https://arxiv.org/abs/2602.03712v1)
**Category:** cs.SE | **Published:** 2026-02-03 | **Skill Score:** 69
**Authors:** Yisen Xu, Jinqiu Yang,  Tse-Hsun...

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

> Large Language Models (LLMs) have recently attracted wide interest for tackling software engineering tasks. In contrast to code generation, refactoring demands precise, semantics-preserving edits that improve program structure, which also makes automated evaluation challenging. However, existing refactoring benchmarks commonly suffer from three shortcomings: limited coverage of refactoring scenarios, the inclusion of instances that mix refactoring with unrelated changes, and insufficient reposit

Refer to the [full paper](https://arxiv.org/abs/2602.03712v1) for detailed methodology.