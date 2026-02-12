---
name: "blind-gods-and-broken"
description: "The evolution of Large Language Models (LLMs) has shifted mobile computing from App-centric interactions to system-level autonomous agents. Implements techniques from the paper 'Blind Gods and Broken Screens: Architecting a Secure, Intent-Centric Mobile Agent Operating System' for extract, transform, and process data. Use when tasks involve (data processing), (agent framework), (prompt engineering), (security), (web automation) or when the user references techniques from this research area."
---

# Blind Gods and Broken Screens: Architecting a Secure, Intent-Centric Mobile Agent Operating System

**Source:** [https://arxiv.org/abs/2602.10915v1](https://arxiv.org/abs/2602.10915v1)
**Category:** cs.CR | **Published:** 2026-02-11 | **Skill Score:** 78
**Authors:** Zhenhua Zou, Sheng Guo, Qiuyang Zhan...

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

> The evolution of Large Language Models (LLMs) has shifted mobile computing from App-centric interactions to system-level autonomous agents. Current implementations predominantly rely on a "Screen-as-Interface" paradigm, which inherits structural vulnerabilities and conflicts with the mobile ecosystem's economic foundations. In this paper, we conduct a systematic security analysis of state-of-the-art mobile agents using Doubao Mobile Assistant as a representative case. We decompose the threat lan

Refer to the [full paper](https://arxiv.org/abs/2602.10915v1) for detailed methodology.