---
name: "malicious-agent-skills-in"
description: "Third-party agent skills extend LLM-based agents with instruction files and executable code that run on users' machines. Implements techniques from the paper 'Malicious Agent Skills in the Wild: A Large-Scale Security Empirical Study' for extract, transform, and process data. Use when tasks involve (data processing), (search & retrieval), (agent framework), (prompt engineering), (security) or when the user references techniques from this research area."
---

# Malicious Agent Skills in the Wild: A Large-Scale Security Empirical Study

**Source:** [https://arxiv.org/abs/2602.06547v1](https://arxiv.org/abs/2602.06547v1)
**Category:** cs.CR | **Published:** 2026-02-06 | **Skill Score:** 72
**Authors:** Yi Liu, Zhihao Chen, Yanjun Zhang...

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

> Third-party agent skills extend LLM-based agents with instruction files and executable code that run on users' machines. Skills execute with user privileges and are distributed through community registries with minimal vetting, but no ground-truth dataset exists to characterize the resulting threats. We construct the first labeled dataset of malicious agent skills by behaviorally verifying 98,380 skills from two community registries, confirming 157 malicious skills with 632 vulnerabilities. Thes

Refer to the [full paper](https://arxiv.org/abs/2602.06547v1) for detailed methodology.