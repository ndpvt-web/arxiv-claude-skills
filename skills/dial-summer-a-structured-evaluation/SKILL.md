---
name: "dial-summer-a-structured-evaluation"
description: "Dialogues are a predominant mode of communication for humans, and it is immensely helpful to have automatically generated summaries of them (e.g., to revise key points discussed in a meeting, to re... Implements techniques from the paper 'DIAL-SUMMER: A Structured Evaluation Framework of Hierarchical Errors in Dialogue Summaries' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework) or when the user references techniques from this research area."
---

# DIAL-SUMMER: A Structured Evaluation Framework of Hierarchical Errors in Dialogue Summaries

**Source:** [https://arxiv.org/abs/2602.08149v1](https://arxiv.org/abs/2602.08149v1)
**Category:** cs.CL | **Published:** 2026-02-08 | **Skill Score:** 67
**Authors:** Sahana Ramnath, Nima Chitsazan, Mingyang Zhou...

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

> Dialogues are a predominant mode of communication for humans, and it is immensely helpful to have automatically generated summaries of them (e.g., to revise key points discussed in a meeting, to review conversations between customer agents and product users). Prior works on dialogue summary evaluation largely ignore the complexities specific to this task: (i) shift in structure, from multiple speakers discussing information in a scattered fashion across several turns, to a summary's sentences, a

Refer to the [full paper](https://arxiv.org/abs/2602.08149v1) for detailed methodology.