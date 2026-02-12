---
name: "ivc-prune-revealing-the-implicit"
description: "Large Vision-Language Models (LVLMs) achieve impressive performance across multiple tasks. Implements techniques from the paper 'IVC-Prune: Revealing the Implicit Visual Coordinates in LVLMs for Vision Token Pruning' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# IVC-Prune: Revealing the Implicit Visual Coordinates in LVLMs for Vision Token Pruning

**Source:** [https://arxiv.org/abs/2602.03060v1](https://arxiv.org/abs/2602.03060v1)
**Category:** cs.CV | **Published:** 2026-02-03 | **Skill Score:** 60
**Authors:** Zhichao Sun, Yidong Ma, Gang Liu...

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

> Large Vision-Language Models (LVLMs) achieve impressive performance across multiple tasks. A significant challenge, however, is their prohibitive inference cost when processing high-resolution visual inputs. While visual token pruning has emerged as a promising solution, existing methods that primarily focus on semantic relevance often discard tokens that are crucial for spatial reasoning. We address this gap through a novel insight into \emph{how LVLMs process spatial reasoning}. Specifically, 

Refer to the [full paper](https://arxiv.org/abs/2602.03060v1) for detailed methodology.