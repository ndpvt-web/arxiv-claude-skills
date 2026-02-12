---
name: "tkg-thinker-towards-dynamic-reasoning"
description: "Temporal knowledge graph question answering (TKGQA) aims to answer time-sensitive questions by leveraging temporal knowledge bases. Implements techniques from the paper 'TKG-Thinker: Towards Dynamic Reasoning over Temporal Knowledge Graphs via Agentic Reinforcement Learning' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# TKG-Thinker: Towards Dynamic Reasoning over Temporal Knowledge Graphs via Agentic Reinforcement Learning

**Source:** [https://arxiv.org/abs/2602.05818v2](https://arxiv.org/abs/2602.05818v2)
**Category:** cs.AI | **Published:** 2026-02-05 | **Skill Score:** 83
**Authors:** Zihao Jiang, Miao Peng, Zhenyan Shan...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Leverages:** temporal knowledge bases

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

> Temporal knowledge graph question answering (TKGQA) aims to answer time-sensitive questions by leveraging temporal knowledge bases. While Large Language Models (LLMs) demonstrate significant potential in TKGQA, current prompting strategies constrain their efficacy in two primary ways. First, they are prone to reasoning hallucinations under complex temporal constraints. Second, static prompting limits model autonomy and generalization, as it lack optimization through dynamic interaction with temp

Refer to the [full paper](https://arxiv.org/abs/2602.05818v2) for detailed methodology.