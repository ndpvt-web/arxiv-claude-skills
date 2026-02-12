---
name: "aicd-bench-a-challenging"
description: "Large language models (LLMs) are increasingly capable of generating functional source code, raising concerns about authorship, accountability, and security. Implements techniques from the paper 'AICD Bench: A Challenging Benchmark for AI-Generated Code Detection' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework), (security) or when the user references techniques from this research area."
---

# AICD Bench: A Challenging Benchmark for AI-Generated Code Detection

**Source:** [https://arxiv.org/abs/2602.02079v1](https://arxiv.org/abs/2602.02079v1)
**Category:** cs.LG | **Published:** 2026-02-02 | **Skill Score:** 79
**Authors:** Daniil Orel, Dilshod Azizov, Indraneil Paul...

## Core Capability

Build and orchestrate AI agent workflows.

## Key Techniques

- **Proposed technique:** $\emph{aicd bench}$

## Workflow

1. Decompose complex tasks into subtasks
2. Plan agent coordination and communication patterns
3. Execute subtasks with appropriate tools and models
4. Monitor agent progress and handle failures
5. Aggregate results and present final output

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

> Large language models (LLMs) are increasingly capable of generating functional source code, raising concerns about authorship, accountability, and security. While detecting AI-generated code is critical, existing datasets and benchmarks are narrow, typically limited to binary human-machine classification under in-distribution settings. To bridge this gap, we introduce $\emph{AICD Bench}$, the most comprehensive benchmark for AI-generated code detection. It spans $\emph{2M examples}$, $\emph{77 m

Refer to the [full paper](https://arxiv.org/abs/2602.02079v1) for detailed methodology.