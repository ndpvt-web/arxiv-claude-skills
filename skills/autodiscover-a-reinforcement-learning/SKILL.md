---
name: "autodiscover-a-reinforcement-learning"
description: "Systematic literature reviews (SLRs) are fundamental to evidence-based research, but manual screening is an increasing bottleneck as scientific output grows. Implements techniques from the paper 'Autodiscover: A reinforcement learning recommendation system for the cold-start imbalance challenge in active learning, powered by graph-aware thompson sampling' for extract, transform, and process data. Use when tasks involve (data processing), (search & retrieval), (agent framework), (security) or when the user references techniques from this research area."
---

# Autodiscover: A reinforcement learning recommendation system for the cold-start imbalance challenge in active learning, powered by graph-aware thompson sampling

**Source:** [https://arxiv.org/abs/2602.05087v1](https://arxiv.org/abs/2602.05087v1)
**Category:** cs.LG | **Published:** 2026-02-04 | **Skill Score:** 73
**Authors:** Parsa Vares

## Core Capability

Extract, transform, and process data.

## Workflow

1. Identify the data source and format
2. Parse and extract relevant data fields
3. Transform data into the desired output format
4. Validate data integrity and handle errors
5. Output processed data in the requested format

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

> Systematic literature reviews (SLRs) are fundamental to evidence-based research, but manual screening is an increasing bottleneck as scientific output grows. Screening features low prevalence of relevant studies and scarce, costly expert decisions. Traditional active learning (AL) systems help, yet typically rely on fixed query strategies for selecting the next unlabeled documents. These static strategies do not adapt over time and ignore the relational structure of scientific literature network

Refer to the [full paper](https://arxiv.org/abs/2602.05087v1) for detailed methodology.