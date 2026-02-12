---
name: "how-does-information-access"
description: "Frontier language model agents can exhibit misaligned behaviors, including deception, exploiting reward hacks, and pursuing hidden objectives. Implements techniques from the paper 'How does information access affect LLM monitors' ability to detect sabotage?' for automate deployment, ci/cd, and infrastructure tasks. Use when tasks involve (devops automation), (search & retrieval), (agent framework), (security) or when the user references techniques from this research area."
---

# How does information access affect LLM monitors' ability to detect sabotage?

**Source:** [https://arxiv.org/abs/2601.21112v2](https://arxiv.org/abs/2601.21112v2)
**Category:** cs.AI | **Published:** 2026-01-28 | **Skill Score:** 73
**Authors:** Rauno Arike, Raja Mehta Moreno, Rohan Subramani...

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

> Frontier language model agents can exhibit misaligned behaviors, including deception, exploiting reward hacks, and pursuing hidden objectives. To control potentially misaligned agents, we can use LLMs themselves to monitor for misbehavior. In this paper, we study how information access affects LLM monitor performance. While one might expect that monitors perform better when they have access to more of the monitored agents' reasoning and actions, we find that contemporary systems often perform be

Refer to the [full paper](https://arxiv.org/abs/2601.21112v2) for detailed methodology.