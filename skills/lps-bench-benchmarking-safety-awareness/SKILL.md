---
name: "lps-bench-benchmarking-safety-awareness"
description: "Computer-use agents (CUAs) that interact with real computer systems can perform automated tasks but face critical safety risks. Implements techniques from the paper 'LPS-Bench: Benchmarking Safety Awareness of Computer-Use Agents in Long-Horizon Planning under Benign and Adversarial Scenarios' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# LPS-Bench: Benchmarking Safety Awareness of Computer-Use Agents in Long-Horizon Planning under Benign and Adversarial Scenarios

**Source:** [https://arxiv.org/abs/2602.03255v1](https://arxiv.org/abs/2602.03255v1)
**Category:** cs.AI | **Published:** 2026-02-03 | **Skill Score:** 70
**Authors:** Tianyu Chen, Chujia Hu, Ge Gao...

## Core Capability

Build and orchestrate AI agent workflows.

## Workflow

1. Decompose complex tasks into subtasks
2. Plan agent coordination and communication patterns
3. Execute subtasks with appropriate tools and models
4. Monitor agent progress and handle failures
5. Aggregate results and present final output

## Agent Coordination

- Decompose complex tasks into independent subtasks where possible
- Use parallel execution for independent subtasks
- Implement error recovery and retry logic
- Aggregate partial results when full completion fails

## Research Context

> Computer-use agents (CUAs) that interact with real computer systems can perform automated tasks but face critical safety risks. Ambiguous instructions may trigger harmful actions, and adversarial users can manipulate tool execution to achieve malicious goals. Existing benchmarks mostly focus on short-horizon or GUI-based tasks, evaluating on execution-time errors but overlooking the ability to anticipate planning-time risks. To fill this gap, we present LPS-Bench, a benchmark that evaluates the 

Refer to the [full paper](https://arxiv.org/abs/2602.03255v1) for detailed methodology.