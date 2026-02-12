---
name: "cost-aware-model-selection-for"
description: "Large language models (LLMs) such as GPT-4o and Claude Sonnet 4.5 have demonstrated strong capabilities in open-ended reasoning and generative language tasks, leading to their widespread adoption a... Implements techniques from the paper 'Cost-Aware Model Selection for Text Classification: Multi-Objective Trade-offs Between Fine-Tuned Encoders and LLM Prompting in Production' for automate deployment, ci/cd, and infrastructure tasks. Use when tasks involve (devops automation), (agent framework), (prompt engineering), (design & ui) or when the user references techniques from this research area."
---

# Cost-Aware Model Selection for Text Classification: Multi-Objective Trade-offs Between Fine-Tuned Encoders and LLM Prompting in Production

**Source:** [https://arxiv.org/abs/2602.06370v1](https://arxiv.org/abs/2602.06370v1)
**Category:** cs.CL | **Published:** 2026-02-06 | **Skill Score:** 74
**Authors:** Alberto Andres Valdes Gonzalez

## Core Capability

Automate deployment, CI/CD, and infrastructure tasks.

## Key Techniques

- **Proposed technique:** a systematic comparis

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

> Large language models (LLMs) such as GPT-4o and Claude Sonnet 4.5 have demonstrated strong capabilities in open-ended reasoning and generative language tasks, leading to their widespread adoption across a broad range of NLP applications. However, for structured text classification problems with fixed label spaces, model selection is often driven by predictive performance alone, overlooking operational constraints encountered in production systems.   In this work, we present a systematic comparis

Refer to the [full paper](https://arxiv.org/abs/2602.06370v1) for detailed methodology.