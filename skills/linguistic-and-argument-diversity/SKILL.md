---
name: "linguistic-and-argument-diversity"
description: "The construction of function calling agents has emerged as a promising avenue for extending model capabilities. Implements techniques from the paper 'Linguistic and Argument Diversity in Synthetic Data for Function-Calling Agents' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# Linguistic and Argument Diversity in Synthetic Data for Function-Calling Agents

**Source:** [https://arxiv.org/abs/2601.17829v1](https://arxiv.org/abs/2601.17829v1)
**Category:** cs.CL | **Published:** 2026-01-25 | **Skill Score:** 64
**Authors:** Dan Greenstein, Zohar Karnin, Chen Amiraz...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** a method that generates synthetic datasets via optimizing general

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

> The construction of function calling agents has emerged as a promising avenue for extending model capabilities. A major challenge for this task is obtaining high quality diverse data for training. Prior work emphasizes diversity in functions, invocation patterns, and interaction turns, yet linguistic diversity of requests and coverage of arguments (e.g., \texttt{city\_name}, \texttt{stock\_ticker}) remain underexplored. We propose a method that generates synthetic datasets via optimizing general

Refer to the [full paper](https://arxiv.org/abs/2601.17829v1) for detailed methodology.