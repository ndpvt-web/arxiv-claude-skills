---
name: "teaching-and-evaluating-llms"
description: "Research in AI4Science has shown promise in many science applications, including polymer design. Implements techniques from the paper 'Teaching and Evaluating LLMs to Reason About Polymer Design Related Tasks' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# Teaching and Evaluating LLMs to Reason About Polymer Design Related Tasks

**Source:** [https://arxiv.org/abs/2601.16312v1](https://arxiv.org/abs/2601.16312v1)
**Category:** cs.CL | **Published:** 2026-01-22 | **Skill Score:** 68
**Authors:** Dikshya Mohanty, Mohammad Saqib Hasan, Syed Mostofa Monsur...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Leverages:** a knowledge base of 13m+ data poi

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

> Research in AI4Science has shown promise in many science applications, including polymer design. However, current LLMs prove ineffective on this problem space because: (i) most models lack polymer-specific knowledge (ii) existing aligned models lack coverage of knowledge and capabilities relevant to polymer design. Addressing this, we introduce PolyBench, a large scale training and test benchmark dataset of more than 125K polymer design related tasks, leveraging a knowledge base of 13M+ data poi

Refer to the [full paper](https://arxiv.org/abs/2601.16312v1) for detailed methodology.