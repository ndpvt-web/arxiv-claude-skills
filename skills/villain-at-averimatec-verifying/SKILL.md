---
name: "villain-at-averimatec-verifying"
description: "This paper describes VILLAIN, a multimodal fact-checking system that verifies image-text claims through prompt-based multi-agent collaboration. Implements techniques from the paper 'VILLAIN at AVerImaTeC: Verifying Image-Text Claims via Multi-Agent Collaboration' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# VILLAIN at AVerImaTeC: Verifying Image-Text Claims via Multi-Agent Collaboration

**Source:** [https://arxiv.org/abs/2602.04587v1](https://arxiv.org/abs/2602.04587v1)
**Category:** cs.CL | **Published:** 2026-02-04 | **Skill Score:** 75
**Authors:** Jaeyoon Jung, Yejun Yoon, Seunghyun Yoon...

## Core Capability

Build and orchestrate AI agent workflows.

## Key Techniques

- **Multi-agent architecture** for task decomposition and parallel execution

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

> This paper describes VILLAIN, a multimodal fact-checking system that verifies image-text claims through prompt-based multi-agent collaboration. For the AVerImaTeC shared task, VILLAIN employs vision-language model agents across multiple stages of fact-checking. Textual and visual evidence is retrieved from the knowledge store enriched through additional web collection. To identify key information and address inconsistencies among evidence items, modality-specific and cross-modal agents generate 

Refer to the [full paper](https://arxiv.org/abs/2602.04587v1) for detailed methodology.