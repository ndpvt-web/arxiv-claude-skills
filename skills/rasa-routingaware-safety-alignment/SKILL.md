---
name: "rasa-routingaware-safety-alignment"
description: "Mixture-of-Experts (MoE) language models introduce unique challenges for safety alignment due to their sparse routing mechanisms, which can enable degenerate optimization behaviors under standard f... Implements techniques from the paper 'RASA: Routing-Aware Safety Alignment for Mixture-of-Experts Models' for build and orchestrate ai agent workflows. Use when tasks involve (general AI assistance) or when the user references techniques from this research area."
---

# RASA: Routing-Aware Safety Alignment for Mixture-of-Experts Models

**Source:** [https://arxiv.org/abs/2602.04448v1](https://arxiv.org/abs/2602.04448v1)
**Category:** cs.LG | **Published:** 2026-02-04 | **Skill Score:** 61
**Authors:** Jiacheng Liang, Yuhui Wang, Tanqiu Jiang...

## Core Capability

Build and orchestrate AI agent workflows.

## Workflow

1. Decompose complex tasks into subtasks
2. Plan agent coordination and communication patterns
3. Execute subtasks with appropriate tools and models
4. Monitor agent progress and handle failures
5. Aggregate results and present final output

## Research Context

> Mixture-of-Experts (MoE) language models introduce unique challenges for safety alignment due to their sparse routing mechanisms, which can enable degenerate optimization behaviors under standard full-parameter fine-tuning. In our preliminary experiments, we observe that naively applying full-parameter safety fine-tuning to MoE models can reduce attack success rates through routing or expert dominance effects, rather than by directly repairing Safety-Critical Experts. To address this challenge, 

Refer to the [full paper](https://arxiv.org/abs/2602.04448v1) for detailed methodology.