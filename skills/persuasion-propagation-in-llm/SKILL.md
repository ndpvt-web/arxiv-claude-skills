---
name: "persuasion-propagation-in-llm"
description: "Modern AI agents increasingly combine conversational interaction with autonomous task execution, such as coding and web research, raising a natural question: what happens when an agent engaged in l... Implements techniques from the paper 'Persuasion Propagation in LLM Agents' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# Persuasion Propagation in LLM Agents

**Source:** [https://arxiv.org/abs/2602.00851v1](https://arxiv.org/abs/2602.00851v1)
**Category:** cs.AI | **Published:** 2026-01-31 | **Skill Score:** 58
**Authors:** Hyejun Jeong, Amir Houmansadr, Shlomo Zilberstein...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** a behavior-centered evaluation framework that distinguishes between persuasion applied during or prior to ta

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

> Modern AI agents increasingly combine conversational interaction with autonomous task execution, such as coding and web research, raising a natural question: what happens when an agent engaged in long-horizon tasks is subjected to user persuasion? We study how belief-level intervention can influence downstream task behavior, a phenomenon we name \emph{persuasion propagation}. We introduce a behavior-centered evaluation framework that distinguishes between persuasion applied during or prior to ta

Refer to the [full paper](https://arxiv.org/abs/2602.00851v1) for detailed methodology.