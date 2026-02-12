---
name: "ide-bench-evaluating-large-language"
description: "IDE-Bench is a comprehensive framework for evaluating AI IDE agents on real-world software engineering tasks through an IDE-native tool interface. Implements techniques from the paper 'IDE-Bench: Evaluating Large Language Models as IDE Agents on Real-World Software Engineering Tasks' for refactor, migrate, or transform existing code. Use when tasks involve (code transformation), (devops automation), (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# IDE-Bench: Evaluating Large Language Models as IDE Agents on Real-World Software Engineering Tasks

**Source:** [https://arxiv.org/abs/2601.20886v2](https://arxiv.org/abs/2601.20886v2)
**Category:** cs.SE | **Published:** 2026-01-28 | **Skill Score:** 85
**Authors:** Spencer Mateega, Jeff Yang, Tiana Costello...

## Core Capability

Refactor, migrate, or transform existing code.

## Key Techniques

- **Proposed technique:** a dockerized test harness that goes beyond raw terminal execution

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

> IDE-Bench is a comprehensive framework for evaluating AI IDE agents on real-world software engineering tasks through an IDE-native tool interface. We present a Dockerized test harness that goes beyond raw terminal execution, granting models a structured tool ecosystem that represents AI-native IDEs like Cursor and Windsurf. By providing high-level abstractions for codebase search, structured file editing, and tools for testing full-stack applications, IDE-Bench evaluates an agent's ability to ac

Refer to the [full paper](https://arxiv.org/abs/2601.20886v2) for detailed methodology.