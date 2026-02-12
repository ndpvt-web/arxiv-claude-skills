---
name: "featurebench-benchmarking-agentic-coding"
description: "Agents powered by large language models (LLMs) are increasingly adopted in the software industry, contributing code as collaborators or even autonomous developers. Implements techniques from the paper 'FeatureBench: Benchmarking Agentic Coding for Complex Feature Development' for refactor, migrate, or transform existing code. Use when tasks involve (code transformation), (testing), (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# FeatureBench: Benchmarking Agentic Coding for Complex Feature Development

**Source:** [https://arxiv.org/abs/2602.10975v1](https://arxiv.org/abs/2602.10975v1)
**Category:** cs.SE | **Published:** 2026-02-11 | **Skill Score:** 89
**Authors:** Qixing Zhou, Jiacheng Zhang, Haiyang Wang...

## Core Capability

Refactor, migrate, or transform existing code.

## Workflow

1. Understand the current code structure and dependencies
2. Plan the transformation strategy (refactor, migrate, translate)
3. Apply transformations while preserving functionality
4. Verify correctness through testing
5. Document the changes made

## Testing Approach

- Generate unit tests covering happy path and edge cases
- Include boundary value tests
- Test error handling paths
- Aim for high code coverage

## Agent Coordination

- Decompose complex tasks into independent subtasks where possible
- Use parallel execution for independent subtasks
- Implement error recovery and retry logic
- Aggregate partial results when full completion fails

## Research Context

> Agents powered by large language models (LLMs) are increasingly adopted in the software industry, contributing code as collaborators or even autonomous developers. As their presence grows, it becomes important to assess the current boundaries of their coding abilities. Existing agentic coding benchmarks, however, cover a limited task scope, e.g., bug fixing within a single pull request (PR), and often rely on non-executable evaluations or lack an automated approach for continually updating the e

Refer to the [full paper](https://arxiv.org/abs/2602.10975v1) for detailed methodology.