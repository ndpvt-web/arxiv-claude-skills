---
name: "naamse-framework-for-evolutionary"
description: "AI agents are increasingly deployed in production, yet their security evaluations remain bottlenecked by manual red-teaming or static benchmarks that fail to model adaptive, multi-turn adversaries. Implements techniques from the paper 'NAAMSE: Framework for Evolutionary Security Evaluation of Agents' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework), (prompt engineering), (security) or when the user references techniques from this research area."
---

# NAAMSE: Framework for Evolutionary Security Evaluation of Agents

**Source:** [https://arxiv.org/abs/2602.07391v1](https://arxiv.org/abs/2602.07391v1)
**Category:** cs.AI | **Published:** 2026-02-07 | **Skill Score:** 75
**Authors:** Kunal Pai, Parth Shah, Harshil Patel

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

> AI agents are increasingly deployed in production, yet their security evaluations remain bottlenecked by manual red-teaming or static benchmarks that fail to model adaptive, multi-turn adversaries. We propose NAAMSE, an evolutionary framework that reframes agent security evaluation as a feedback-driven optimization problem. Our system employs a single autonomous agent that orchestrates a lifecycle of genetic prompt mutation, hierarchical corpus exploration, and asymmetric behavioral scoring. By 

Refer to the [full paper](https://arxiv.org/abs/2602.07391v1) for detailed methodology.