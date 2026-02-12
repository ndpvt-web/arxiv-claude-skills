---
name: "metagen-selfevolving-roles-and"
description: "Large language models are increasingly deployed as multi-agent systems, where specialized roles communicate and collaborate through structured interactions to solve complex tasks that often exceed ... Implements techniques from the paper 'MetaGen: Self-Evolving Roles and Topologies for Multi-Agent LLM Reasoning' for generate code from natural language descriptions. Use when tasks involve (code generation), (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# MetaGen: Self-Evolving Roles and Topologies for Multi-Agent LLM Reasoning

**Source:** [https://arxiv.org/abs/2601.19290v1](https://arxiv.org/abs/2601.19290v1)
**Category:** cs.CL | **Published:** 2026-01-27 | **Skill Score:** 71
**Authors:** Yimeng Wang, Jiaxing Zhao, Hongbin Xie...

## Core Capability

Generate code from natural language descriptions.

## Key Techniques

- **Achievement:** the capacity of a single agent
- **Multi-agent architecture** for task decomposition and parallel execution

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

> Large language models are increasingly deployed as multi-agent systems, where specialized roles communicate and collaborate through structured interactions to solve complex tasks that often exceed the capacity of a single agent. However, most existing systems still rely on a fixed role library and an execution-frozen interaction topology, a rigid design choice that frequently leads to task mismatch, prevents timely adaptation when new evidence emerges during reasoning, and further inflates infer

Refer to the [full paper](https://arxiv.org/abs/2601.19290v1) for detailed methodology.