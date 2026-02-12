---
name: "stepshield-when-not-whether"
description: "Existing agent safety benchmarks report binary accuracy, conflating early intervention with post-mortem analysis. Implements techniques from the paper 'StepShield: When, Not Whether to Intervene on Rogue Agents' for automate deployment, ci/cd, and infrastructure tasks. Use when tasks involve (devops automation), (agent framework), (security) or when the user references techniques from this research area."
---

# StepShield: When, Not Whether to Intervene on Rogue Agents

**Source:** [https://arxiv.org/abs/2601.22136v1](https://arxiv.org/abs/2601.22136v1)
**Category:** cs.LG | **Published:** 2026-01-29 | **Skill Score:** 75
**Authors:** Gloria Felicia, Michael Eniolade, Jinfeng He...

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

> Existing agent safety benchmarks report binary accuracy, conflating early intervention with post-mortem analysis. A detector that flags a violation at step 8 enables intervention; one that reports it at step 48 provides only forensic value. This distinction is critical, yet current benchmarks cannot measure it. We introduce StepShield, the first benchmark to evaluate when violations are detected, not just whether. StepShield contains 9,213 code agent trajectories, including 1,278 meticulously an

Refer to the [full paper](https://arxiv.org/abs/2601.22136v1) for detailed methodology.