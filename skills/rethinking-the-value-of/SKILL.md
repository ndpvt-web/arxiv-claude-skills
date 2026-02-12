---
name: "rethinking-the-value-of"
description: "Large Language Model (LLM) code agents increasingly resolve repository-level issues by iteratively editing code, invoking tools, and validating candidate patches. Implements techniques from the paper 'Rethinking the Value of Agent-Generated Tests for LLM-Based Software Engineering Agents' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# Rethinking the Value of Agent-Generated Tests for LLM-Based Software Engineering Agents

**Source:** [https://arxiv.org/abs/2602.07900v1](https://arxiv.org/abs/2602.07900v1)
**Category:** cs.SE | **Published:** 2026-02-08 | **Skill Score:** 66
**Authors:** Zhi Chen, Zhensu Sun, Yuling Shi...

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

> Large Language Model (LLM) code agents increasingly resolve repository-level issues by iteratively editing code, invoking tools, and validating candidate patches. In these workflows, agents often write tests on the fly, a paradigm adopted by many high-ranking agents on the SWE-bench leaderboard. However, we observe that GPT-5.2, which writes almost no new tests, can even achieve performance comparable to top-ranking agents. This raises the critical question: whether such tests meaningfully impro

Refer to the [full paper](https://arxiv.org/abs/2602.07900v1) for detailed methodology.