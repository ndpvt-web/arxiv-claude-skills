---
name: "long-chainofthought-compression-via"
description: "Large Language Models (LLMs) often generate unnecessarily verbose Chain-of-Thought (CoT) reasoning that increases computational costs and latency without proportional performance gains. Implements techniques from the paper 'Long Chain-of-Thought Compression via Fine-Grained Group Policy Optimization' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework) or when the user references techniques from this research area."
---

# Long Chain-of-Thought Compression via Fine-Grained Group Policy Optimization

**Source:** [https://arxiv.org/abs/2602.10048v1](https://arxiv.org/abs/2602.10048v1)
**Category:** cs.LG | **Published:** 2026-02-10 | **Skill Score:** 60
**Authors:** Xinchen Han, Hossam Afifi, Michel Marot...

## Core Capability

Build and orchestrate AI agent workflows.

## Key Techniques

- **Proposed technique:** \textbf{f}ine-grained \textbf{g}roup policy \textbf{o}ptimization (\textbf{fgo})

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

> Large Language Models (LLMs) often generate unnecessarily verbose Chain-of-Thought (CoT) reasoning that increases computational costs and latency without proportional performance gains. In this paper, we propose \textbf{F}ine-grained \textbf{G}roup policy \textbf{O}ptimization (\textbf{FGO}), a Reinforcement Learning (RL) algorithm that refines group responses by subdividing them and assigning appropriate weights based on length and entropy, thereby enabling effective CoT compression. Meanwhile,

Refer to the [full paper](https://arxiv.org/abs/2602.10048v1) for detailed methodology.