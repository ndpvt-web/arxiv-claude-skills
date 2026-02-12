---
name: "you-need-an-encoder"
description: "The Key-Value (KV) cache of Large Language Models (LLMs) is prefix-based, making it highly inefficient for processing contexts retrieved in arbitrary order. Implements techniques from the paper 'You Need an Encoder for Native Position-Independent Caching' for build and orchestrate ai agent workflows. Use when tasks involve (general AI assistance) or when the user references techniques from this research area."
---

# You Need an Encoder for Native Position-Independent Caching

**Source:** [https://arxiv.org/abs/2602.01519v1](https://arxiv.org/abs/2602.01519v1)
**Category:** cs.LG | **Published:** 2026-02-02 | **Skill Score:** 63
**Authors:** Shiju Zhao, Junhao Hu, Jiaqi Zheng...

## Core Capability

Build and orchestrate AI agent workflows.

## Key Techniques

- **Proposed technique:** native pic by reintroducing the encoder to prevalent decoder-only llms and explicitly training

## Workflow

1. Decompose complex tasks into subtasks
2. Plan agent coordination and communication patterns
3. Execute subtasks with appropriate tools and models
4. Monitor agent progress and handle failures
5. Aggregate results and present final output

## Research Context

> The Key-Value (KV) cache of Large Language Models (LLMs) is prefix-based, making it highly inefficient for processing contexts retrieved in arbitrary order. Position-Independent Caching (PIC) has been proposed to enable KV reuse without positional constraints; however, existing approaches often incur substantial accuracy degradation, limiting their practical adoption. To address this issue, we propose native PIC by reintroducing the encoder to prevalent decoder-only LLMs and explicitly training 

Refer to the [full paper](https://arxiv.org/abs/2602.01519v1) for detailed methodology.