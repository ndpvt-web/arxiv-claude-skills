---
name: "algoveri-an-aligned-benchmark"
description: "Vericoding refers to the generation of formally verified code from rigorous specifications. Implements techniques from the paper 'AlgoVeri: An Aligned Benchmark for Verified Code Generation on Classical Algorithms' for generate code from natural language descriptions. Use when tasks involve (code generation) or when the user references techniques from this research area."
---

# AlgoVeri: An Aligned Benchmark for Verified Code Generation on Classical Algorithms

**Source:** [https://arxiv.org/abs/2602.09464v1](https://arxiv.org/abs/2602.09464v1)
**Category:** cs.SE | **Published:** 2026-02-10 | **Skill Score:** 92
**Authors:** Haoyu Zhao, Ziran Yang, Jiawei Li...

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

## Research Context

> Vericoding refers to the generation of formally verified code from rigorous specifications. Recent AI models show promise in vericoding, but a unified methodology for cross-paradigm evaluation is lacking. Existing benchmarks test only individual languages/tools (e.g., Dafny, Verus, and Lean) and each covers very different tasks, so the performance numbers are not directly comparable. We address this gap with AlgoVeri, a benchmark that evaluates vericoding of $77$ classical algorithms in Dafny, V

Refer to the [full paper](https://arxiv.org/abs/2602.09464v1) for detailed methodology.