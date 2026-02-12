---
name: "surgery-mitigating-harmful-finetuning"
description: "Harmful fine-tuning can invalidate safety alignment of large language models, exposing significant safety risks. Implements techniques from the paper 'Surgery: Mitigating Harmful Fine-Tuning for Large Language Models via Attention Sink' for build and orchestrate ai agent workflows. Use when tasks involve (general AI assistance) or when the user references techniques from this research area."
---

# Surgery: Mitigating Harmful Fine-Tuning for Large Language Models via Attention Sink

**Source:** [https://arxiv.org/abs/2602.05228v2](https://arxiv.org/abs/2602.05228v2)
**Category:** cs.AI | **Published:** 2026-02-05 | **Skill Score:** 59
**Authors:** Guozhi Liu, Weiwei Lin, Tiansheng Huang...

## Core Capability

Build and orchestrate AI agent workflows.

## Key Techniques

- **Leverages:** the attention sink mechanism to mitigate harmful fine-tuning

## Workflow

1. Decompose complex tasks into subtasks
2. Plan agent coordination and communication patterns
3. Execute subtasks with appropriate tools and models
4. Monitor agent progress and handle failures
5. Aggregate results and present final output

## Research Context

> Harmful fine-tuning can invalidate safety alignment of large language models, exposing significant safety risks. In this paper, we utilize the attention sink mechanism to mitigate harmful fine-tuning. Specifically, we first measure a statistic named \emph{sink divergence} for each attention head and observe that \emph{different attention heads exhibit two different signs of sink divergence}. To understand its safety implications, we conduct experiments and find that the number of attention heads

Refer to the [full paper](https://arxiv.org/abs/2602.05228v2) for detailed methodology.