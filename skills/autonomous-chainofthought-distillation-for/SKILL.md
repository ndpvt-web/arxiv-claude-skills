---
name: "autonomous-chainofthought-distillation-for"
description: "Graph-based fraud detection on text-attributed graphs (TAGs) requires jointly modeling rich textual semantics and relational dependencies. Implements techniques from the paper 'Autonomous Chain-of-Thought Distillation for Graph-Based Fraud Detection' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# Autonomous Chain-of-Thought Distillation for Graph-Based Fraud Detection

**Source:** [https://arxiv.org/abs/2601.22949v1](https://arxiv.org/abs/2601.22949v1)
**Category:** cs.CL | **Published:** 2026-01-30 | **Skill Score:** 79
**Authors:** Yuan Li, Jun Hu, Bryan Hooi...

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

> Graph-based fraud detection on text-attributed graphs (TAGs) requires jointly modeling rich textual semantics and relational dependencies. However, existing LLM-enhanced GNN approaches are constrained by predefined prompting and decoupled training pipelines, limiting reasoning autonomy and weakening semantic-structural alignment. We propose FraudCoT, a unified framework that advances TAG-based fraud detection through autonomous, graph-aware chain-of-thought (CoT) reasoning and scalable LLM-GNN c

Refer to the [full paper](https://arxiv.org/abs/2601.22949v1) for detailed methodology.