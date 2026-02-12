---
name: "spd-faith-bench-diagnosing-and"
description: "Chain-of-Thought reasoning is widely used to improve the interpretability of multimodal large language models (MLLMs), yet the faithfulness of the generated reasoning traces remains unclear. Implements techniques from the paper 'SPD-Faith Bench: Diagnosing and Improving Faithfulness in Chain-of-Thought for Multimodal Large Language Models' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework) or when the user references techniques from this research area."
---

# SPD-Faith Bench: Diagnosing and Improving Faithfulness in Chain-of-Thought for Multimodal Large Language Models

**Source:** [https://arxiv.org/abs/2602.07833v1](https://arxiv.org/abs/2602.07833v1)
**Category:** cs.CV | **Published:** 2026-02-08 | **Skill Score:** 83
**Authors:** Weijiang Lv, Yaoxuan Feng, Xiaobo Xia...

## Core Capability

Build and orchestrate AI agent workflows.

## Key Techniques

- **Proposed technique:** spd-faith bench

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

> Chain-of-Thought reasoning is widely used to improve the interpretability of multimodal large language models (MLLMs), yet the faithfulness of the generated reasoning traces remains unclear. Prior work has mainly focused on perceptual hallucinations, leaving reasoning level unfaithfulness underexplored. To isolate faithfulness from linguistic priors, we introduce SPD-Faith Bench, a diagnostic benchmark based on fine-grained image difference reasoning that enforces explicit visual comparison. Eva

Refer to the [full paper](https://arxiv.org/abs/2602.07833v1) for detailed methodology.