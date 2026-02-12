---
name: "scaling-agentic-verifier-for"
description: "Large language models (LLMs) have demonstrated strong coding capabilities but still struggle to solve competitive programming problems correctly in a single attempt. Implements techniques from the paper 'Scaling Agentic Verifier for Competitive Coding' for generate and manage test suites. Use when tasks involve (testing), (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# Scaling Agentic Verifier for Competitive Coding

**Source:** [https://arxiv.org/abs/2602.04254v1](https://arxiv.org/abs/2602.04254v1)
**Category:** cs.CL | **Published:** 2026-02-04 | **Skill Score:** 63
**Authors:** Zeyao Ma, Jing Zhang, Xiaokang Zhang...

## Core Capability

Generate and manage test suites.

## Key Techniques

- **Proposed technique:** agentic verifier

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

> Large language models (LLMs) have demonstrated strong coding capabilities but still struggle to solve competitive programming problems correctly in a single attempt. Execution-based re-ranking offers a promising test-time scaling strategy, yet existing methods are constrained by either difficult test case generation or inefficient random input sampling. To address this limitation, we propose Agentic Verifier, an execution-based agent that actively reasons about program behaviors and searches for

Refer to the [full paper](https://arxiv.org/abs/2602.04254v1) for detailed methodology.