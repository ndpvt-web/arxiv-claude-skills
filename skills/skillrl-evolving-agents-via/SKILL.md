---
name: "skillrl-evolving-agents-via"
description: "Large Language Model (LLM) agents have shown stunning results in complex tasks, yet they often operate in isolation, failing to learn from past experiences. Implements techniques from the paper 'SkillRL: Evolving Agents via Recursive Skill-Augmented Reinforcement Learning' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# SkillRL: Evolving Agents via Recursive Skill-Augmented Reinforcement Learning

**Source:** [https://arxiv.org/abs/2602.08234v1](https://arxiv.org/abs/2602.08234v1)
**Category:** cs.LG | **Published:** 2026-02-09 | **Skill Score:** 68
**Authors:** Peng Xia, Jianwen Chen, Hanyang Wang...

## Core Capability

Search, retrieve, and synthesize information.

## Workflow

1. Parse the user's information need or query
2. Formulate effective search strategies
3. Retrieve relevant documents, code, or data
4. Synthesize findings into a coherent response
5. Provide citations and references

## Agent Coordination

- Decompose complex tasks into independent subtasks where possible
- Use parallel execution for independent subtasks
- Implement error recovery and retry logic
- Aggregate partial results when full completion fails

## Research Context

> Large Language Model (LLM) agents have shown stunning results in complex tasks, yet they often operate in isolation, failing to learn from past experiences. Existing memory-based methods primarily store raw trajectories, which are often redundant and noise-heavy. This prevents agents from extracting high-level, reusable behavioral patterns that are essential for generalization. In this paper, we propose SkillRL, a framework that bridges the gap between raw experience and policy improvement throu

Refer to the [full paper](https://arxiv.org/abs/2602.08234v1) for detailed methodology.