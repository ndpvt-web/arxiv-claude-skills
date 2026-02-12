---
name: "magic-a-coevolving-attackerdefender"
description: "Ensuring robust safety alignment is crucial for Large Language Models (LLMs), yet existing defenses often lag behind evolving adversarial attacks due to their \textbf{reliance on static, pre-collec... Implements techniques from the paper 'MAGIC: A Co-Evolving Attacker-Defender Adversarial Game for Robust LLM Safety' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# MAGIC: A Co-Evolving Attacker-Defender Adversarial Game for Robust LLM Safety

**Source:** [https://arxiv.org/abs/2602.01539v2](https://arxiv.org/abs/2602.01539v2)
**Category:** cs.AI | **Published:** 2026-02-02 | **Skill Score:** 93
**Authors:** Xiaoyu Wen, Zhida He, Han Qi...

## Core Capability

Build and orchestrate AI agent workflows.

## Key Techniques

- **Proposed technique:** \textbf{magic}
- **Novel approach:** multi-turn multi-agent reinforcement learning framework
- **Multi-agent architecture** for task decomposition and parallel execution

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

> Ensuring robust safety alignment is crucial for Large Language Models (LLMs), yet existing defenses often lag behind evolving adversarial attacks due to their \textbf{reliance on static, pre-collected data distributions}. In this paper, we introduce \textbf{MAGIC}, a novel multi-turn multi-agent reinforcement learning framework that formulates LLM safety alignment as an adversarial asymmetric game. Specifically, an attacker agent learns to iteratively rewrite original queries into deceptive prom

Refer to the [full paper](https://arxiv.org/abs/2602.01539v2) for detailed methodology.