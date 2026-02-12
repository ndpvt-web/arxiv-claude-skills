---
name: "scaling-mobile-chaos-testing"
description: "Mobile applications in large-scale distributed systems are susceptible to backend service failures, yet traditional chaos engineering approaches cannot scale mobile testing due to the combinatorial... Implements techniques from the paper 'Scaling Mobile Chaos Testing with AI-Driven Test Execution' for generate and manage test suites. Use when tasks involve (testing), (search & retrieval) or when the user references techniques from this research area."
---

# Scaling Mobile Chaos Testing with AI-Driven Test Execution

**Source:** [https://arxiv.org/abs/2602.06223v1](https://arxiv.org/abs/2602.06223v1)
**Category:** cs.SE | **Published:** 2026-02-05 | **Skill Score:** 60
**Authors:** Juan Marcano, Ashish Samant, Kai Song...

## Core Capability

Generate and manage test suites.

## Key Techniques

- **Proposed technique:** an automated mobile chaos testing system that integrates dragoncrawl

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

## Research Context

> Mobile applications in large-scale distributed systems are susceptible to backend service failures, yet traditional chaos engineering approaches cannot scale mobile testing due to the combinatorial explosion of flows, locations, and failure scenarios that need validation. We present an automated mobile chaos testing system that integrates DragonCrawl, an LLM-based mobile testing platform, with uHavoc, a service-level fault injection system. The key insight is that adaptive AI-driven test executi

Refer to the [full paper](https://arxiv.org/abs/2602.06223v1) for detailed methodology.