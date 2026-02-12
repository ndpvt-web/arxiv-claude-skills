---
name: "fullstack-agent-enhancing-agentic-fullstack"
description: "Assisting non-expert users to develop complex interactive websites has become a popular task for LLM-powered code agents. Implements techniques from the paper 'FullStack-Agent: Enhancing Agentic Full-Stack Web Coding via Development-Oriented Testing and Repository Back-Translation' for generate and manage test suites. Use when tasks involve (testing), (search & retrieval), (agent framework), (database & query), (design & ui) or when the user references techniques from this research area."
---

# FullStack-Agent: Enhancing Agentic Full-Stack Web Coding via Development-Oriented Testing and Repository Back-Translation

**Source:** [https://arxiv.org/abs/2602.03798v1](https://arxiv.org/abs/2602.03798v1)
**Category:** cs.SE | **Published:** 2026-02-03 | **Skill Score:** 100
**Authors:** Zimu Lu, Houxing Ren, Yunqiao Yang...

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

> Assisting non-expert users to develop complex interactive websites has become a popular task for LLM-powered code agents. However, existing code agents tend to only generate frontend web pages, masking the lack of real full-stack data processing and storage with fancy visual effects. Notably, constructing production-level full-stack web applications is far more challenging than only generating frontend web pages, demanding careful control of data flow, comprehensive understanding of constantly u

Refer to the [full paper](https://arxiv.org/abs/2602.03798v1) for detailed methodology.