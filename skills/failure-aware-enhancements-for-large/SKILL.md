---
name: "failure-aware-enhancements-for-large"
description: "Large language models (LLMs) show promise for automating software development by translating requirements into code. Implements techniques from the paper 'Failure-Aware Enhancements for Large Language Model (LLM) Code Generation: An Empirical Study on Decision Framework' for generate code from natural language descriptions. Use when tasks involve (code generation), (search & retrieval), (prompt engineering) or when the user references techniques from this research area."
---

# Failure-Aware Enhancements for Large Language Model (LLM) Code Generation: An Empirical Study on Decision Framework

**Source:** [https://arxiv.org/abs/2602.02896v1](https://arxiv.org/abs/2602.02896v1)
**Category:** cs.SE | **Published:** 2026-02-02 | **Skill Score:** 67
**Authors:** Jianru Shen, Zedong Peng, Lucy Owen

## Core Capability

Generate code from natural language descriptions.

## Key Techniques

- **Retrieval-augmented** approach for grounding responses in external knowledge

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

> Large language models (LLMs) show promise for automating software development by translating requirements into code. However, even advanced prompting workflows like progressive prompting often leave some requirements unmet. Although methods such as self-critique, multi-model collaboration, and retrieval-augmented generation (RAG) have been proposed to address these gaps, developers lack clear guidance on when to use each. In an empirical study of 25 GitHub projects, we found that progressive pro

Refer to the [full paper](https://arxiv.org/abs/2602.02896v1) for detailed methodology.