---
name: "focus-session-llm4pqc-an"
description: "The design of post-quantum cryptography (PQC) hardware is a complex and hierarchical process with many challenges. Implements techniques from the paper 'Focus Session: LLM4PQC -- An Agentic Framework for Accurate and Efficient Synthesis of PQC Cores' for refactor, migrate, or transform existing code. Use when tasks involve (code transformation), (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# Focus Session: LLM4PQC -- An Agentic Framework for Accurate and Efficient Synthesis of PQC Cores

**Source:** [https://arxiv.org/abs/2602.09919v1](https://arxiv.org/abs/2602.09919v1)
**Category:** cs.CR | **Published:** 2026-02-10 | **Skill Score:** 65
**Authors:** Buddhi Perera, Zeng Wang, Weihua Xiao...

## Core Capability

Refactor, migrate, or transform existing code.

## Workflow

1. Understand the current code structure and dependencies
2. Plan the transformation strategy (refactor, migrate, translate)
3. Apply transformations while preserving functionality
4. Verify correctness through testing
5. Document the changes made

## Agent Coordination

- Decompose complex tasks into independent subtasks where possible
- Use parallel execution for independent subtasks
- Implement error recovery and retry logic
- Aggregate partial results when full completion fails

## Research Context

> The design of post-quantum cryptography (PQC) hardware is a complex and hierarchical process with many challenges. A primary bottleneck is the conversion of PQC reference codes from C to high-level synthesis (HLS) specifications, which requires extensive manual refactoring [1]-[3]. Another bottleneck is the scalability of synthesis for complex PQC primitives, including number theoretic transform (NTT) accelerators and wide memory interfaces. While large language models (LLMs) have shown remarkab

Refer to the [full paper](https://arxiv.org/abs/2602.09919v1) for detailed methodology.