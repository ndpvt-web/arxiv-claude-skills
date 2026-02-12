---
name: "authenticated-workflows-a-systems"
description: "Agentic AI systems automate enterprise workflows but existing defenses--guardrails, semantic filters--are probabilistic and routinely bypassed. Implements techniques from the paper 'Authenticated Workflows: A Systems Approach to Protecting Agentic AI' for generate and manage test suites. Use when tasks involve (testing), (agent framework), (prompt engineering), (security) or when the user references techniques from this research area."
---

# Authenticated Workflows: A Systems Approach to Protecting Agentic AI

**Source:** [https://arxiv.org/abs/2602.10465v1](https://arxiv.org/abs/2602.10465v1)
**Category:** cs.CR | **Published:** 2026-02-11 | **Skill Score:** 61
**Authors:** Mohan Rajagopalan, Vinay Rao

## Core Capability

Generate and manage test suites.

## Key Techniques

- **Proposed technique:** authenticated workflows

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

## Security Considerations

- Check for OWASP Top 10 vulnerabilities
- Validate all user inputs at system boundaries
- Use parameterized queries for database access
- Follow the principle of least privilege

## Agent Coordination

- Decompose complex tasks into independent subtasks where possible
- Use parallel execution for independent subtasks
- Implement error recovery and retry logic
- Aggregate partial results when full completion fails

## Research Context

> Agentic AI systems automate enterprise workflows but existing defenses--guardrails, semantic filters--are probabilistic and routinely bypassed. We introduce authenticated workflows, the first complete trust layer for enterprise agentic AI. Security reduces to protecting four fundamental boundaries: prompts, tools, data, and context. We enforce intent (operations satisfy organizational policies) and integrity (operations are cryptographically authentic) at every boundary crossing, combining crypt

Refer to the [full paper](https://arxiv.org/abs/2602.10465v1) for detailed methodology.