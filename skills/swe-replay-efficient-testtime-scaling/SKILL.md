---
name: "swe-replay-efficient-testtime-scaling"
description: "Test-time scaling has been widely adopted to enhance the capabilities of Large Language Model (LLM) agents in software engineering (SWE) tasks. Implements techniques from the paper 'SWE-Replay: Efficient Test-Time Scaling for Software Engineering Agents' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework), (security) or when the user references techniques from this research area."
---

# SWE-Replay: Efficient Test-Time Scaling for Software Engineering Agents

**Source:** [https://arxiv.org/abs/2601.22129v2](https://arxiv.org/abs/2601.22129v2)
**Category:** cs.SE | **Published:** 2026-01-29 | **Skill Score:** 80
**Authors:** Yifeng Ding, Lingming Zhang

## Core Capability

Build and orchestrate AI agent workflows.

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

> Test-time scaling has been widely adopted to enhance the capabilities of Large Language Model (LLM) agents in software engineering (SWE) tasks. However, the standard approach of repeatedly sampling trajectories from scratch is computationally expensive. While recent methods have attempted to mitigate costs using specialized value agents, they can suffer from model miscalibration and fail to generalize to modern agents that synthesize custom bash scripts as tools. In this paper, we introduce SWE-

Refer to the [full paper](https://arxiv.org/abs/2601.22129v2) for detailed methodology.