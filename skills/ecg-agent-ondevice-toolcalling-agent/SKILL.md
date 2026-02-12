---
name: "ecg-agent-ondevice-toolcalling-agent"
description: "Recent advances in Multimodal Large Language Models have rapidly expanded to electrocardiograms, focusing on classification, report generation, and single-turn QA tasks. Implements techniques from the paper 'ECG-Agent: On-Device Tool-Calling Agent for ECG Multi-Turn Dialogue' for automate deployment, ci/cd, and infrastructure tasks. Use when tasks involve (devops automation), (agent framework) or when the user references techniques from this research area."
---

# ECG-Agent: On-Device Tool-Calling Agent for ECG Multi-Turn Dialogue

**Source:** [https://arxiv.org/abs/2601.20323v1](https://arxiv.org/abs/2601.20323v1)
**Category:** cs.AI | **Published:** 2026-01-28 | **Skill Score:** 67
**Authors:** Hyunseung Chung, Jungwoo Oh, Daeun Kyung...

## Core Capability

Automate deployment, CI/CD, and infrastructure tasks.

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

> Recent advances in Multimodal Large Language Models have rapidly expanded to electrocardiograms, focusing on classification, report generation, and single-turn QA tasks. However, these models fall short in real-world scenarios, lacking multi-turn conversational ability, on-device efficiency, and precise understanding of ECG measurements such as the PQRST intervals. To address these limitations, we introduce ECG-Agent, the first LLM-based tool-calling agent for multi-turn ECG dialogue. To facilit

Refer to the [full paper](https://arxiv.org/abs/2601.20323v1) for detailed methodology.