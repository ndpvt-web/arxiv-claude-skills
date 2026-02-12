---
name: "sd-e2-semantic-exploration-for-reasoning-under"
description: "Small language models (SLMs) struggle with complex reasoning because exploration is expensive under tight compute budgets. Implements techniques from the paper 'SD-E$^2$: Semantic Exploration for Reasoning Under Token Budgets' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework), (security) or when the user references techniques from this research area."
---

# SD-E$^2$: Semantic Exploration for Reasoning Under Token Budgets

**Source:** [https://arxiv.org/abs/2601.17982v1](https://arxiv.org/abs/2601.17982v1)
**Category:** cs.CL | **Published:** 2026-01-25 | **Skill Score:** 65
**Authors:** Kshitij Mishra, Nils Lukas, Salem Lahlou

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** semantic diversity-exploration-exploitation (sd-e$^2$)

## Workflow

1. Parse the user's information need or query
2. Formulate effective search strategies
3. Retrieve relevant documents, code, or data
4. Synthesize findings into a coherent response
5. Provide citations and references

## Security Considerations

- Check for OWASP Top 10 vulnerabilities
- Validate all user inputs at system boundaries
- Use parameterized queries for database access
- Follow the principle of least privilege

## Agent Coordination

- Decompose complex tasks into independent subtasks where possible
- Use parallel execution for independent subtasks
- Implement error recovery and retry logic
- Aggregate partial results when full completion fails

## Research Context

> Small language models (SLMs) struggle with complex reasoning because exploration is expensive under tight compute budgets. We introduce Semantic Diversity-Exploration-Exploitation (SD-E$^2$), a reinforcement learning framework that makes exploration explicit by optimizing semantic diversity in generated reasoning trajectories. Using a frozen sentence-embedding model, SD-E$^2$ assigns a diversity reward that captures (i) the coverage of semantically distinct solution strategies and (ii) their ave

Refer to the [full paper](https://arxiv.org/abs/2601.17982v1) for detailed methodology.