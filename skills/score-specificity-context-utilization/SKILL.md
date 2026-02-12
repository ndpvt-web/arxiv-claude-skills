---
name: "score-specificity-context-utilization"
description: "Large language models (LLMs) are increasingly used to support question answering and decision-making in high-stakes, domain-specific settings such as natural hazard response and infrastructure plan... Implements techniques from the paper 'SCORE: Specificity, Context Utilization, Robustness, and Relevance for Reference-Free LLM Evaluation' for automate deployment, ci/cd, and infrastructure tasks. Use when tasks involve (devops automation), (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# SCORE: Specificity, Context Utilization, Robustness, and Relevance for Reference-Free LLM Evaluation

**Source:** [https://arxiv.org/abs/2602.10017v1](https://arxiv.org/abs/2602.10017v1)
**Category:** cs.CL | **Published:** 2026-02-10 | **Skill Score:** 62
**Authors:** Homaira Huda Shomee, Rochana Chaturvedi, Yangxinyu Xie...

## Core Capability

Automate deployment, CI/CD, and infrastructure tasks.

## Key Techniques

- **Retrieval-augmented** approach for grounding responses in external knowledge

## Workflow

1. Analyze the current infrastructure and deployment setup
2. Design automation workflows for the target environment
3. Generate configuration files (Docker, K8s, CI/CD pipelines)
4. Implement monitoring and alerting
5. Validate the automation works correctly

## Agent Coordination

- Decompose complex tasks into independent subtasks where possible
- Use parallel execution for independent subtasks
- Implement error recovery and retry logic
- Aggregate partial results when full completion fails

## Research Context

> Large language models (LLMs) are increasingly used to support question answering and decision-making in high-stakes, domain-specific settings such as natural hazard response and infrastructure planning, where effective answers must convey fine-grained, decision-critical details. However, existing evaluation frameworks for retrieval-augmented generation (RAG) and open-ended question answering primarily rely on surface-level similarity, factual consistency, or semantic relevance, and often fail to

Refer to the [full paper](https://arxiv.org/abs/2602.10017v1) for detailed methodology.