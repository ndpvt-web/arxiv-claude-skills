---
name: "how-to-build-ai"
description: "Critical domain knowledge typically resides with few experts, creating organizational bottlenecks in scalability and decision-making. Implements techniques from the paper 'How to Build AI Agents by Augmenting LLMs with Codified Human Expert Domain Knowledge? A Software Engineering Framework' for generate code from natural language descriptions. Use when tasks involve (code generation), (code analysis), (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# How to Build AI Agents by Augmenting LLMs with Codified Human Expert Domain Knowledge? A Software Engineering Framework

**Source:** [https://arxiv.org/abs/2601.15153v1](https://arxiv.org/abs/2601.15153v1)
**Category:** cs.AI | **Published:** 2026-01-21 | **Skill Score:** 66
**Authors:** Choro Ulan uulu, Mikhail Kulyabin, Iris Fuhrmann...

## Core Capability

Generate code from natural language descriptions.

## Key Techniques

- **Proposed technique:** a software engineering framework to capture human domain knowledge for engineering ai agents in simulation data

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

> Critical domain knowledge typically resides with few experts, creating organizational bottlenecks in scalability and decision-making. Non-experts struggle to create effective visualizations, leading to suboptimal insights and diverting expert time. This paper investigates how to capture and embed human domain knowledge into AI agent systems through an industrial case study. We propose a software engineering framework to capture human domain knowledge for engineering AI agents in simulation data 

Refer to the [full paper](https://arxiv.org/abs/2601.15153v1) for detailed methodology.