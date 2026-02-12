---
name: "cognitively-diverse-multiplechoice-question"
description: "Recent advances in large language models (LLMs) have made automated multiple-choice question (MCQ) generation increasingly feasible; however, reliably producing items that satisfy controlled cognit... Implements techniques from the paper 'Cognitively Diverse Multiple-Choice Question Generation: A Hybrid Multi-Agent Framework with Large Language Models' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework), (prompt engineering), (design & ui) or when the user references techniques from this research area."
---

# Cognitively Diverse Multiple-Choice Question Generation: A Hybrid Multi-Agent Framework with Large Language Models

**Source:** [https://arxiv.org/abs/2602.03704v1](https://arxiv.org/abs/2602.03704v1)
**Category:** cs.CL | **Published:** 2026-02-03 | **Skill Score:** 77
**Authors:** Yu Tian, Linh Huynh, Katerina Christhilf...

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

> Recent advances in large language models (LLMs) have made automated multiple-choice question (MCQ) generation increasingly feasible; however, reliably producing items that satisfy controlled cognitive demands remains a challenge. To address this gap, we introduce ReQUESTA, a hybrid, multi-agent framework for generating cognitively diverse MCQs that systematically target text-based, inferential, and main idea comprehension. ReQUESTA decomposes MCQ authoring into specialized subtasks and coordinat

Refer to the [full paper](https://arxiv.org/abs/2602.03704v1) for detailed methodology.