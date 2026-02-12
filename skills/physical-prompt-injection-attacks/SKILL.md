---
name: "physical-prompt-injection-attacks"
description: "Large Vision-Language Models (LVLMs) are increasingly deployed in real-world intelligent systems for perception and reasoning in open physical environments. Implements techniques from the paper 'Physical Prompt Injection Attacks on Large Vision-Language Models' for automate deployment, ci/cd, and infrastructure tasks. Use when tasks involve (devops automation), (search & retrieval), (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# Physical Prompt Injection Attacks on Large Vision-Language Models

**Source:** [https://arxiv.org/abs/2601.17383v1](https://arxiv.org/abs/2601.17383v1)
**Category:** cs.CV | **Published:** 2026-01-24 | **Skill Score:** 73
**Authors:** Chen Ling, Kai Hu, Hangcheng Liu...

## Core Capability

Automate deployment, CI/CD, and infrastructure tasks.

## Key Techniques

- **Proposed technique:** the first physical prompt injection attack (ppia)

## Workflow

1. Analyze the current infrastructure and deployment setup
2. Design automation workflows for the target environment
3. Generate configuration files (Docker, K8s, CI/CD pipelines)
4. Implement monitoring and alerting
5. Validate the automation works correctly

## Agent Coordination

- Decompose complex tasks into independent subtasks where possible
- Use parallel execution for independent subtasks
- Implement error recovery and retry logic
- Aggregate partial results when full completion fails

## Research Context

> Large Vision-Language Models (LVLMs) are increasingly deployed in real-world intelligent systems for perception and reasoning in open physical environments. While LVLMs are known to be vulnerable to prompt injection attacks, existing methods either require access to input channels or depend on knowledge of user queries, assumptions that rarely hold in practical deployments. We propose the first Physical Prompt Injection Attack (PPIA), a black-box, query-agnostic attack that embeds malicious typo

Refer to the [full paper](https://arxiv.org/abs/2601.17383v1) for detailed methodology.