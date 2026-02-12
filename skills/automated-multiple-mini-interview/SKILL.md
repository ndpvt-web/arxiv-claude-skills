---
name: "automated-multiple-mini-interview"
description: "Assessing soft skills such as empathy, ethical judgment, and communication is essential in competitive selection processes, yet human scoring is often inconsistent and biased. Implements techniques from the paper 'Automated Multiple Mini Interview (MMI) Scoring' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# Automated Multiple Mini Interview (MMI) Scoring

**Source:** [https://arxiv.org/abs/2602.02360v1](https://arxiv.org/abs/2602.02360v1)
**Category:** cs.CL | **Published:** 2026-02-02 | **Skill Score:** 77
**Authors:** Ryan Huynh, Frank Guerin, Alison Callwood

## Core Capability

Build and orchestrate AI agent workflows.

## Key Techniques

- **Proposed technique:** a multi-agent
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

> Assessing soft skills such as empathy, ethical judgment, and communication is essential in competitive selection processes, yet human scoring is often inconsistent and biased. While Large Language Models (LLMs) have improved Automated Essay Scoring (AES), we show that state-of-the-art rationale-based fine-tuning methods struggle with the abstract, context-dependent nature of Multiple Mini-Interviews (MMIs), missing the implicit signals embedded in candidate narratives. We introduce a multi-agent

Refer to the [full paper](https://arxiv.org/abs/2602.02360v1) for detailed methodology.