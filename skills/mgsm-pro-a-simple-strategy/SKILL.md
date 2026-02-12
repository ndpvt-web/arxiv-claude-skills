---
name: "mgsm-pro-a-simple-strategy"
description: "Large language models have made substantial progress in mathematical reasoning. Implements techniques from the paper 'MGSM-Pro: A Simple Strategy for Robust Multilingual Mathematical Reasoning Evaluation' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework) or when the user references techniques from this research area."
---

# MGSM-Pro: A Simple Strategy for Robust Multilingual Mathematical Reasoning Evaluation

**Source:** [https://arxiv.org/abs/2601.21225v1](https://arxiv.org/abs/2601.21225v1)
**Category:** cs.CL | **Published:** 2026-01-29 | **Skill Score:** 63
**Authors:** Tianyi Xu, Kosei Uemura, Alfred Malengo Kondoro...

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

> Large language models have made substantial progress in mathematical reasoning. However, benchmark development for multilingual evaluation has lagged behind English in both difficulty and recency. Recently, GSM-Symbolic showed a strong evidence of high variance when models are evaluated on different instantiations of the same question; however, the evaluation was conducted only in English. In this paper, we introduce MGSM-Pro, an extension of MGSM dataset with GSM-Symbolic approach. Our dataset 

Refer to the [full paper](https://arxiv.org/abs/2601.21225v1) for detailed methodology.