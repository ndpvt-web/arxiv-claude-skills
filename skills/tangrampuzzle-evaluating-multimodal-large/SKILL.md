---
name: "tangrampuzzle-evaluating-multimodal-large"
description: "Multimodal Large Language Models (MLLMs) have achieved remarkable progress in visual recognition and semantic understanding. Implements techniques from the paper 'TangramPuzzle: Evaluating Multimodal Large Language Models with Compositional Spatial Reasoning' for generate code from natural language descriptions. Use when tasks involve (code generation), (agent framework), (design & ui) or when the user references techniques from this research area."
---

# TangramPuzzle: Evaluating Multimodal Large Language Models with Compositional Spatial Reasoning

**Source:** [https://arxiv.org/abs/2601.16520v1](https://arxiv.org/abs/2601.16520v1)
**Category:** cs.CV | **Published:** 2026-01-23 | **Skill Score:** 83
**Authors:** Daixian Liu, Jiayi Kuang, Yinghui Li...

## Core Capability

Generate code from natural language descriptions.

## Key Techniques

- **Proposed technique:** tangrampuzz

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

> Multimodal Large Language Models (MLLMs) have achieved remarkable progress in visual recognition and semantic understanding. Nevertheless, their ability to perform precise compositional spatial reasoning remains largely unexplored. Existing benchmarks often involve relatively simple tasks and rely on semantic approximations or coarse relative positioning, while their evaluation metrics are typically limited and lack rigorous mathematical formulations. To bridge this gap, we introduce TangramPuzz

Refer to the [full paper](https://arxiv.org/abs/2601.16520v1) for detailed methodology.