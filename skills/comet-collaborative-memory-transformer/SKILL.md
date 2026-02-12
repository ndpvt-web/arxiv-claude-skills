---
name: "comet-collaborative-memory-transformer"
description: "The quadratic complexity and indefinitely growing key-value (KV) cache of standard Transformers pose a major barrier to long-context processing. Implements techniques from the paper 'CoMeT: Collaborative Memory Transformer for Efficient Long Context Modeling' for generate and maintain documentation. Use when tasks involve (documentation), (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# CoMeT: Collaborative Memory Transformer for Efficient Long Context Modeling

**Source:** [https://arxiv.org/abs/2602.01766v1](https://arxiv.org/abs/2602.01766v1)
**Category:** cs.LG | **Published:** 2026-02-02 | **Skill Score:** 83
**Authors:** Runsong Zhao, Shilei Liu, Jiwei Tang...

## Core Capability

Generate and maintain documentation.

## Key Techniques

- **Proposed technique:** the collaborative memory transformer (comet)

## Workflow

1. Analyze the codebase to understand architecture and APIs
2. Generate clear, accurate documentation in the appropriate format
3. Include code examples and usage patterns
4. Maintain consistency with existing documentation style

## Agent Coordination

- Decompose complex tasks into independent subtasks where possible
- Use parallel execution for independent subtasks
- Implement error recovery and retry logic
- Aggregate partial results when full completion fails

## Research Context

> The quadratic complexity and indefinitely growing key-value (KV) cache of standard Transformers pose a major barrier to long-context processing. To overcome this, we introduce the Collaborative Memory Transformer (CoMeT), a novel architecture that enables LLMs to handle arbitrarily long sequences with constant memory usage and linear time complexity. Designed as an efficient, plug-in module, CoMeT can be integrated into pre-trained models with only minimal fine-tuning. It operates on sequential 

Refer to the [full paper](https://arxiv.org/abs/2602.01766v1) for detailed methodology.