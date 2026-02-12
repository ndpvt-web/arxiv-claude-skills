---
name: "agenticpay-a-multiagent-llm"
description: "Large language model (LLM)-based agents are increasingly expected to negotiate, coordinate, and transact autonomously, yet existing benchmarks lack principled settings for evaluating language-media... Implements techniques from the paper 'AgenticPay: A Multi-Agent LLM Negotiation System for Buyer-Seller Transactions' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework) or when the user references techniques from this research area."
---

# AgenticPay: A Multi-Agent LLM Negotiation System for Buyer-Seller Transactions

**Source:** [https://arxiv.org/abs/2602.06008v1](https://arxiv.org/abs/2602.06008v1)
**Category:** cs.AI | **Published:** 2026-02-05 | **Skill Score:** 81
**Authors:** Xianyang Liu, Shangding Gu, Dawn Song

## Core Capability

Build and orchestrate AI agent workflows.

## Key Techniques

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

> Large language model (LLM)-based agents are increasingly expected to negotiate, coordinate, and transact autonomously, yet existing benchmarks lack principled settings for evaluating language-mediated economic interaction among multiple agents. We introduce AgenticPay, a benchmark and simulation framework for multi-agent buyer-seller negotiation driven by natural language. AgenticPay models markets in which buyers and sellers possess private constraints and product-dependent valuations, and must

Refer to the [full paper](https://arxiv.org/abs/2602.06008v1) for detailed methodology.