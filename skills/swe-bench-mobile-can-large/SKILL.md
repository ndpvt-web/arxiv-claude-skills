---
name: "swe-bench-mobile-can-large"
description: "Can large language model agents develop industry-level mobile applications? We introduce \textbf{SWE-Bench Mobile}, a benchmark for evaluating coding agents on realistic software engineering tasks ... Implements techniques from the paper 'SWE-Bench Mobile: Can Large Language Model Agents Develop Industry-Level Mobile Applications?' for refactor, migrate, or transform existing code. Use when tasks involve (code transformation), (devops automation), (search & retrieval), (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# SWE-Bench Mobile: Can Large Language Model Agents Develop Industry-Level Mobile Applications?

**Source:** [https://arxiv.org/abs/2602.09540v1](https://arxiv.org/abs/2602.09540v1)
**Category:** cs.SE | **Published:** 2026-02-10 | **Skill Score:** 71
**Authors:** Muxin Tian, Zhe Wang, Blair Yang...

## Core Capability

Refactor, migrate, or transform existing code.

## Key Techniques

- **Proposed technique:** \textbf{swe-bench mobile}

## Workflow

1. Understand the current code structure and dependencies
2. Plan the transformation strategy (refactor, migrate, translate)
3. Apply transformations while preserving functionality
4. Verify correctness through testing
5. Document the changes made

## Agent Coordination

- Decompose complex tasks into independent subtasks where possible
- Use parallel execution for independent subtasks
- Implement error recovery and retry logic
- Aggregate partial results when full completion fails

## Research Context

> Can large language model agents develop industry-level mobile applications? We introduce \textbf{SWE-Bench Mobile}, a benchmark for evaluating coding agents on realistic software engineering tasks derived from a production iOS codebase. Unlike existing benchmarks that focus on isolated problems or bug fixes, SWE-Bench Mobile captures the full complexity of industrial development: multi-modal inputs (PRDs and Figma designs), a large-scale mixed Swift/Objective-C codebase, and comprehensive test s

Refer to the [full paper](https://arxiv.org/abs/2602.09540v1) for detailed methodology.