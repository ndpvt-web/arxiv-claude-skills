---
name: "rift-reordered-instruction-following"
description: "Large Language Models (LLMs) are increasingly relied upon for complex workflows, yet their ability to maintain flow of instructions remains underexplored. Implements techniques from the paper 'RIFT: Reordered Instruction Following Testbed To Evaluate Instruction Following in Singular Multistep Prompt Structures' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# RIFT: Reordered Instruction Following Testbed To Evaluate Instruction Following in Singular Multistep Prompt Structures

**Source:** [https://arxiv.org/abs/2601.18924v1](https://arxiv.org/abs/2601.18924v1)
**Category:** cs.AI | **Published:** 2026-01-26 | **Skill Score:** 65
**Authors:** Andrew Jaffe, Noah Reicin, Jinho D. Choi

## Core Capability

Build and orchestrate AI agent workflows.

## Workflow

1. Decompose complex tasks into subtasks
2. Plan agent coordination and communication patterns
3. Execute subtasks with appropriate tools and models
4. Monitor agent progress and handle failures
5. Aggregate results and present final output

## Agent Coordination

- Decompose complex tasks into independent subtasks where possible
- Use parallel execution for independent subtasks
- Implement error recovery and retry logic
- Aggregate partial results when full completion fails

## Research Context

> Large Language Models (LLMs) are increasingly relied upon for complex workflows, yet their ability to maintain flow of instructions remains underexplored. Existing benchmarks conflate task complexity with structural ordering, making it difficult to isolate the impact of prompt topology on performance. We introduce RIFT, Reordered Instruction Following Testbed, to assess instruction following by disentangling structure from content. Using rephrased Jeopardy! question-answer pairs, we test LLMs ac

Refer to the [full paper](https://arxiv.org/abs/2601.18924v1) for detailed methodology.