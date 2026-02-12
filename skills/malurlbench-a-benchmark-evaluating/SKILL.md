---
name: "malurlbench-a-benchmark-evaluating"
description: "LLM-based web agents have become increasingly popular for their utility in daily life and work. Implements techniques from the paper 'MalURLBench: A Benchmark Evaluating Agents' Vulnerabilities When Processing Web URLs' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework), (security) or when the user references techniques from this research area."
---

# MalURLBench: A Benchmark Evaluating Agents' Vulnerabilities When Processing Web URLs

**Source:** [https://arxiv.org/abs/2601.18113v2](https://arxiv.org/abs/2601.18113v2)
**Category:** cs.CR | **Published:** 2026-01-26 | **Skill Score:** 60
**Authors:** Dezhang Kong, Zhuxi Wu, Shiqi Liu...

## Core Capability

Build and orchestrate AI agent workflows.

## Key Techniques

- **Proposed technique:** malurlbench

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

> LLM-based web agents have become increasingly popular for their utility in daily life and work. However, they exhibit critical vulnerabilities when processing malicious URLs: accepting a disguised malicious URL enables subsequent access to unsafe webpages, which can cause severe damage to service providers and users. Despite this risk, no benchmark currently targets this emerging threat. To address this gap, we propose MalURLBench, the first benchmark for evaluating LLMs' vulnerabilities to mali

Refer to the [full paper](https://arxiv.org/abs/2601.18113v2) for detailed methodology.