---
name: "evocodebench-a-humanperformance-benchmark"
description: "As large language models (LLMs) continue to advance in programming tasks, LLM-driven coding systems have evolved from one-shot code generation into complex systems capable of iterative improvement ... Implements techniques from the paper 'EvoCodeBench: A Human-Performance Benchmark for Self-Evolving LLM-Driven Coding Systems' for generate code from natural language descriptions. Use when tasks involve (code generation), (agent framework) or when the user references techniques from this research area."
---

# EvoCodeBench: A Human-Performance Benchmark for Self-Evolving LLM-Driven Coding Systems

**Source:** [https://arxiv.org/abs/2602.10171v1](https://arxiv.org/abs/2602.10171v1)
**Category:** cs.SE | **Published:** 2026-02-10 | **Skill Score:** 75
**Authors:** Wentao Zhang, Jianfeng Wang, Liheng Liang...

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

> As large language models (LLMs) continue to advance in programming tasks, LLM-driven coding systems have evolved from one-shot code generation into complex systems capable of iterative improvement during inference. However, existing code benchmarks primarily emphasize static correctness and implicitly assume fixed model capability during inference. As a result, they do not capture inference-time self-evolution, such as whether accuracy and efficiency improve as an agent iteratively refines its s

Refer to the [full paper](https://arxiv.org/abs/2602.10171v1) for detailed methodology.