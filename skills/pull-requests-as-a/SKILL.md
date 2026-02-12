---
name: "pull-requests-as-a"
description: "Repository-level code editing requires models to understand complex dependencies and execute precise multi-file modifications across a large codebase. Implements techniques from the paper 'Pull Requests as a Training Signal for Repo-Level Code Editing' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# Pull Requests as a Training Signal for Repo-Level Code Editing

**Source:** [https://arxiv.org/abs/2602.07457v1](https://arxiv.org/abs/2602.07457v1)
**Category:** cs.SE | **Published:** 2026-02-07 | **Skill Score:** 72
**Authors:** Qinglin Zhu, Tianyu Chen, Shuai Lu...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** clean pull request (clean-pr)
- **Leverages:** real-world github pull requests as a training signal for repository-level

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

> Repository-level code editing requires models to understand complex dependencies and execute precise multi-file modifications across a large codebase. While recent gains on SWE-bench rely heavily on complex agent scaffolding, it remains unclear how much of this capability can be internalised via high-quality training signals. To address this, we propose Clean Pull Request (Clean-PR), a mid-training paradigm that leverages real-world GitHub pull requests as a training signal for repository-level 

Refer to the [full paper](https://arxiv.org/abs/2602.07457v1) for detailed methodology.