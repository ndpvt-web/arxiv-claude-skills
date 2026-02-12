---
name: "robustexplain-evaluating-robustness-of"
description: "Large Language Models (LLMs) are increasingly used to generate natural-language explanations in recommender systems, acting as explanation agents that reason over user behavior histories. Implements techniques from the paper 'RobustExplain: Evaluating Robustness of LLM-Based Explanation Agents for Recommendation' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework) or when the user references techniques from this research area."
---

# RobustExplain: Evaluating Robustness of LLM-Based Explanation Agents for Recommendation

**Source:** [https://arxiv.org/abs/2601.19120v3](https://arxiv.org/abs/2601.19120v3)
**Category:** cs.IR | **Published:** 2026-01-27 | **Skill Score:** 75
**Authors:** Guilin Zhang, Kai Zhao, Jeffrey Friedman...

## Core Capability

Build and orchestrate AI agent workflows.

## Workflow

1. Decompose complex tasks into subtasks
2. Plan agent coordination and communication patterns
3. Execute subtasks with appropriate tools and models
4. Monitor agent progress and handle failures
5. Aggregate results and present final output

## Agent Coordination

- Decompose complex tasks into independent subtasks where possible
- Use parallel execution for independent subtasks
- Implement error recovery and retry logic
- Aggregate partial results when full completion fails

## Research Context

> Large Language Models (LLMs) are increasingly used to generate natural-language explanations in recommender systems, acting as explanation agents that reason over user behavior histories. While prior work has focused on explanation fluency and relevance under fixed inputs, the robustness of LLM-generated explanations to realistic user behavior noise remains largely unexplored. In real-world web platforms, interaction histories are inherently noisy due to accidental clicks, temporal inconsistenci

Refer to the [full paper](https://arxiv.org/abs/2601.19120v3) for detailed methodology.