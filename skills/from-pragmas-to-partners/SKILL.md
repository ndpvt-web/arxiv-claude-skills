---
name: "from-pragmas-to-partners"
description: "The rise of large language models has sparked interest in AI-driven hardware design, raising the question: does high-level synthesis (HLS) still matter in the agentic era? We argue that HLS remains... Implements techniques from the paper 'From Pragmas to Partners: A Symbiotic Evolution of Agentic High-Level Synthesis' for generate code from natural language descriptions. Use when tasks involve (code generation), (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# From Pragmas to Partners: A Symbiotic Evolution of Agentic High-Level Synthesis

**Source:** [https://arxiv.org/abs/2602.01401v3](https://arxiv.org/abs/2602.01401v3)
**Category:** cs.CL | **Published:** 2026-02-01 | **Skill Score:** 69
**Authors:** Niansong Zhang, Sunwoo Kim, Shreesha Srinath...

## Core Capability

Generate code from natural language descriptions.

## Key Techniques

- **Leverages:** both hls and rtl

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

> The rise of large language models has sparked interest in AI-driven hardware design, raising the question: does high-level synthesis (HLS) still matter in the agentic era? We argue that HLS remains essential. While we expect mature agentic hardware systems to leverage both HLS and RTL, this paper focuses on HLS and its role in enabling agentic optimization. HLS offers faster iteration cycles, portability, and design permutability that make it a natural layer for agentic optimization. This positi

Refer to the [full paper](https://arxiv.org/abs/2602.01401v3) for detailed methodology.