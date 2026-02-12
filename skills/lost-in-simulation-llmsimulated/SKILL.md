---
name: "lost-in-simulation-llmsimulated"
description: "Agentic benchmarks increasingly rely on LLM-simulated users to scalably evaluate agent performance, yet the robustness, validity, and fairness of this approach remain unexamined. Implements techniques from the paper 'Lost in Simulation: LLM-Simulated Users are Unreliable Proxies for Human Users in Agentic Evaluations' for automate deployment, ci/cd, and infrastructure tasks. Use when tasks involve (devops automation), (agent framework) or when the user references techniques from this research area."
---

# Lost in Simulation: LLM-Simulated Users are Unreliable Proxies for Human Users in Agentic Evaluations

**Source:** [https://arxiv.org/abs/2601.17087v2](https://arxiv.org/abs/2601.17087v2)
**Category:** cs.HC | **Published:** 2026-01-23 | **Skill Score:** 70
**Authors:** Preethi Seshadri, Samuel Cahyawijaya, Ayomide Odumakinde...

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

> Agentic benchmarks increasingly rely on LLM-simulated users to scalably evaluate agent performance, yet the robustness, validity, and fairness of this approach remain unexamined. Through a user study with participants across the United States, India, Kenya, and Nigeria, we investigate whether LLM-simulated users serve as reliable proxies for real human users in evaluating agents on τ-Bench retail tasks. We find that user simulation lacks robustness, with agent success rates varying up to 9 perce

Refer to the [full paper](https://arxiv.org/abs/2601.17087v2) for detailed methodology.