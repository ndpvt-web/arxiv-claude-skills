---
name: "graph-augmented-reasoning-with-large"
description: "This paper proposes a graph-augmented reasoning framework for tobacco pest and disease management that integrates structured domain knowledge into large language models. Implements techniques from the paper 'Graph-Augmented Reasoning with Large Language Models for Tobacco Pest and Disease Management' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# Graph-Augmented Reasoning with Large Language Models for Tobacco Pest and Disease Management

**Source:** [https://arxiv.org/abs/2602.02635v1](https://arxiv.org/abs/2602.02635v1)
**Category:** cs.CL | **Published:** 2026-02-02 | **Skill Score:** 65
**Authors:** Siyu Li, Chenwei Song, Qi Zhou...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** a graph-augmented reasoning framework for tobacco pest and disease management that integrates structured domain knowledge into large language models

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

> This paper proposes a graph-augmented reasoning framework for tobacco pest and disease management that integrates structured domain knowledge into large language models. Building on GraphRAG, we construct a domain-specific knowledge graph and retrieve query-relevant subgraphs to provide relational evidence during answer generation. The framework adopts ChatGLM as the Transformer backbone with LoRA-based parameter-efficient fine-tuning, and employs a graph neural network to learn node representat

Refer to the [full paper](https://arxiv.org/abs/2602.02635v1) for detailed methodology.