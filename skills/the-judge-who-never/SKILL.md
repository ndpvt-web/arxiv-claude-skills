---
name: "the-judge-who-never"
description: "Large language models (LLMs) are increasingly used as automatic judges to evaluate system outputs in tasks such as reasoning, question answering, and creative writing. Implements techniques from the paper 'The Judge Who Never Admits: Hidden Shortcuts in LLM-based Evaluation' for automate deployment, ci/cd, and infrastructure tasks. Use when tasks involve (devops automation), (search & retrieval), (content generation), (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# The Judge Who Never Admits: Hidden Shortcuts in LLM-based Evaluation

**Source:** [https://arxiv.org/abs/2602.07996v1](https://arxiv.org/abs/2602.07996v1)
**Category:** cs.CL | **Published:** 2026-02-08 | **Skill Score:** 65
**Authors:** Arash Marioriyad, Omid Ghahroodi, Ehsaneddin Asgari...

## Core Capability

Automate deployment, CI/CD, and infrastructure tasks.

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

> Large language models (LLMs) are increasingly used as automatic judges to evaluate system outputs in tasks such as reasoning, question answering, and creative writing. A faithful judge should base its verdicts solely on content quality, remain invariant to irrelevant context, and transparently reflect the factors driving its decisions. We test this ideal via controlled cue perturbations-synthetic metadata labels injected into evaluation prompts-for six judge models: GPT-4o, Gemini-2.0-Flash, Gem

Refer to the [full paper](https://arxiv.org/abs/2602.07996v1) for detailed methodology.