---
name: "rapo-riskaware-preference-optimization"
description: "Large Reasoning Models (LRMs) have achieved tremendous success with their chain-of-thought (CoT) reasoning, yet also face safety issues similar to those of basic language models. Implements techniques from the paper 'RAPO: Risk-Aware Preference Optimization for Generalizable Safe Reasoning' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# RAPO: Risk-Aware Preference Optimization for Generalizable Safe Reasoning

**Source:** [https://arxiv.org/abs/2602.04224v1](https://arxiv.org/abs/2602.04224v1)
**Category:** cs.LG | **Published:** 2026-02-04 | **Skill Score:** 85
**Authors:** Zeming Wei, Qiaosheng Zhang, Xia Hu...

## Core Capability

Build and orchestrate AI agent workflows.

## Key Techniques

- **Chain-of-thought reasoning** for improved step-by-step problem solving

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

> Large Reasoning Models (LRMs) have achieved tremendous success with their chain-of-thought (CoT) reasoning, yet also face safety issues similar to those of basic language models. In particular, while algorithms are designed to guide them to deliberately refuse harmful prompts with safe reasoning, this process often fails to generalize against diverse and complex jailbreak attacks. In this work, we attribute these failures to the generalization of the safe reasoning process, particularly their in

Refer to the [full paper](https://arxiv.org/abs/2602.04224v1) for detailed methodology.