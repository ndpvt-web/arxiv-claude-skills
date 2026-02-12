---
name: "cleaner-selfpurified-trajectories-boost"
description: "Agentic Reinforcement Learning (RL) has empowered Large Language Models (LLMs) to utilize tools like Python interpreters for complex problem-solving. Implements techniques from the paper 'CLEANER: Self-Purified Trajectories Boost Agentic Reinforcement Learning' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework), (security) or when the user references techniques from this research area."
---

# CLEANER: Self-Purified Trajectories Boost Agentic Reinforcement Learning

**Source:** [https://arxiv.org/abs/2601.15141v1](https://arxiv.org/abs/2601.15141v1)
**Category:** cs.LG | **Published:** 2026-01-21 | **Skill Score:** 66
**Authors:** Tianshi Xu, Yuteng Chen, Meng Li

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Leverages:** tools like python interpreters for complex problem-solving

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

> Agentic Reinforcement Learning (RL) has empowered Large Language Models (LLMs) to utilize tools like Python interpreters for complex problem-solving. However, for parameter-constrained models (e.g., 4B--7B), the exploration phase is often plagued by frequent execution failures, creating noisy trajectories that hinder policy optimization. Under standard outcome-based reward settings, this noise leads to a critical credit assignment issue, where erroneous actions are inadvertently reinforced along

Refer to the [full paper](https://arxiv.org/abs/2601.15141v1) for detailed methodology.