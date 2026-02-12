---
name: "meetbench-xl-calibrated-multidimensional-evaluation"
description: "Enterprise meeting environments require AI assistants that handle diverse operational tasks, from rapid fact checking during live discussions to cross meeting analysis for strategic planning, under... Implements techniques from the paper 'MeetBench-XL: Calibrated Multi-Dimensional Evaluation and Learned Dual-Policy Agents for Real-Time Meetings' for automate deployment, ci/cd, and infrastructure tasks. Use when tasks involve (devops automation), (search & retrieval), (agent framework), (security) or when the user references techniques from this research area."
---

# MeetBench-XL: Calibrated Multi-Dimensional Evaluation and Learned Dual-Policy Agents for Real-Time Meetings

**Source:** [https://arxiv.org/abs/2602.03285v1](https://arxiv.org/abs/2602.03285v1)
**Category:** cs.AI | **Published:** 2026-02-03 | **Skill Score:** 100
**Authors:** Yuelin Hu, Jun Xu, Bingcong Lu...

## Core Capability

Automate deployment, CI/CD, and infrastructure tasks.

## Workflow

1. Analyze the current infrastructure and deployment setup
2. Design automation workflows for the target environment
3. Generate configuration files (Docker, K8s, CI/CD pipelines)
4. Implement monitoring and alerting
5. Validate the automation works correctly

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

> Enterprise meeting environments require AI assistants that handle diverse operational tasks, from rapid fact checking during live discussions to cross meeting analysis for strategic planning, under strict latency, cost, and privacy constraints. Existing meeting benchmarks mainly focus on simplified question answering and fail to reflect real world enterprise workflows, where queries arise organically from multi stakeholder collaboration, span long temporal contexts, and require tool augmented re

Refer to the [full paper](https://arxiv.org/abs/2602.03285v1) for detailed methodology.