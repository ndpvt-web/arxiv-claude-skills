---
name: "testexplora-benchmarking-llms-for"
description: "Given that Large Language Models (LLMs) are increasingly applied to automate software development, comprehensive software assurance spans three distinct goals: regression prevention, reactive repro... Implements techniques from the paper 'TestExplora: Benchmarking LLMs for Proactive Bug Discovery via Repository-Level Test Generation' for generate and manage test suites. Use when tasks involve (testing), (data processing), (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# TestExplora: Benchmarking LLMs for Proactive Bug Discovery via Repository-Level Test Generation

**Source:** [https://arxiv.org/abs/2602.10471v1](https://arxiv.org/abs/2602.10471v1)
**Category:** cs.SE | **Published:** 2026-02-11 | **Skill Score:** 82
**Authors:** Steven Liu, Jane Luo, Xin Zhang...

## Core Capability

Generate and manage test suites.

## Workflow

1. Analyze the code under test to understand its behavior
2. Identify edge cases, boundary conditions, and error paths
3. Generate comprehensive test cases with assertions
4. Run tests and report results
5. Suggest improvements for test coverage

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

> Given that Large Language Models (LLMs) are increasingly applied to automate software development, comprehensive software assurance spans three distinct goals: regression prevention, reactive reproduction, and proactive discovery. Current evaluations systematically overlook the third goal. Specifically, they either treat existing code as ground truth (a compliance trap) for regression prevention, or depend on post-failure artifacts (e.g., issue reports) for bug reproduction-so they rarely surfac

Refer to the [full paper](https://arxiv.org/abs/2602.10471v1) for detailed methodology.