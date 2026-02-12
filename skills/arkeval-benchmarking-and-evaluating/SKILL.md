---
name: "arkeval-benchmarking-and-evaluating"
description: "Large language models have transformed code generation, enabling unprecedented automation in software development. Implements techniques from the paper 'ArkEval: Benchmarking and Evaluating Automated CodeRepair for ArkTS' for generate code from natural language descriptions. Use when tasks involve (code generation), (code transformation), (testing), (search & retrieval) or when the user references techniques from this research area."
---

# ArkEval: Benchmarking and Evaluating Automated CodeRepair for ArkTS

**Source:** [https://arxiv.org/abs/2602.08866v1](https://arxiv.org/abs/2602.08866v1)
**Category:** cs.SE | **Published:** 2026-02-09 | **Skill Score:** 81
**Authors:** Bang Xie, Senjian Zhang, Zhiyuan Peng...

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

## Testing Approach

- Generate unit tests covering happy path and edge cases
- Include boundary value tests
- Test error handling paths
- Aim for high code coverage

## Research Context

> Large language models have transformed code generation, enabling unprecedented automation in software development. As mobile ecosystems evolve, HarmonyOS has emerged as a critical platform requiring robust development tools. Software development for the HarmonyOS ecosystem relies heavily on ArkTS, a statically typed extension of TypeScript. Despite its growing importance, the ecosystem lacks robust tools for automated code repair, primarily due to the absence of a high-quality benchmark for eval

Refer to the [full paper](https://arxiv.org/abs/2602.08866v1) for detailed methodology.