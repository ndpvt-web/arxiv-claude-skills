---
name: "same-stabilized-mixtureofexperts-for"
description: "Multimodal Large Language Models (MLLMs) achieve strong performance through instruction tuning, but real-world deployment requires them to continually expand their capabilities, making Multimodal C... Implements techniques from the paper 'SAME: Stabilized Mixture-of-Experts for Multimodal Continual Instruction Tuning' for automate deployment, ci/cd, and infrastructure tasks. Use when tasks involve (devops automation), (search & retrieval), (prompt engineering) or when the user references techniques from this research area."
---

# SAME: Stabilized Mixture-of-Experts for Multimodal Continual Instruction Tuning

**Source:** [https://arxiv.org/abs/2602.01990v1](https://arxiv.org/abs/2602.01990v1)
**Category:** cs.LG | **Published:** 2026-02-02 | **Skill Score:** 60
**Authors:** Zhen-Hao Xie, Jun-Tao Tang, Yu-Cheng Shi...

## Core Capability

Automate deployment, CI/CD, and infrastructure tasks.

## Key Techniques

- **Leverages:** sparse expert routing to promote task specialization

## Workflow

1. Analyze the current infrastructure and deployment setup
2. Design automation workflows for the target environment
3. Generate configuration files (Docker, K8s, CI/CD pipelines)
4. Implement monitoring and alerting
5. Validate the automation works correctly

## Research Context

> Multimodal Large Language Models (MLLMs) achieve strong performance through instruction tuning, but real-world deployment requires them to continually expand their capabilities, making Multimodal Continual Instruction Tuning (MCIT) essential. Recent methods leverage sparse expert routing to promote task specialization, but we find that the expert routing process suffers from drift as the data distribution evolves. For example, a grounding query that previously activated localization experts may 

Refer to the [full paper](https://arxiv.org/abs/2602.01990v1) for detailed methodology.