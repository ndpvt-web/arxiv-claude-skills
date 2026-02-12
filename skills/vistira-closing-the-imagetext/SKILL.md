---
name: "vistira-closing-the-imagetext"
description: "Vision-language models (VLMs) lag behind text-only language models on mathematical reasoning when the same problems are presented as images rather than text. Implements techniques from the paper 'VisTIRA: Closing the Image-Text Modality Gap in Visual Math Reasoning via Structured Tool Integration' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework), (design & ui) or when the user references techniques from this research area."
---

# VisTIRA: Closing the Image-Text Modality Gap in Visual Math Reasoning via Structured Tool Integration

**Source:** [https://arxiv.org/abs/2601.14440v1](https://arxiv.org/abs/2601.14440v1)
**Category:** cs.AI | **Published:** 2026-01-20 | **Skill Score:** 72
**Authors:** Saeed Khaki, Ashudeep Singh, Nima Safaei...

## Core Capability

Build and orchestrate AI agent workflows.

## Key Techniques

- **Proposed technique:** vistira (vision and tool-integrated reasoning agent)

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

> Vision-language models (VLMs) lag behind text-only language models on mathematical reasoning when the same problems are presented as images rather than text. We empirically characterize this as a modality gap: the same question in text form yields markedly higher accuracy than its visually typeset counterpart, due to compounded failures in reading dense formulas, layout, and mixed symbolic-diagrammatic context. First, we introduce VisTIRA (Vision and Tool-Integrated Reasoning Agent), a tool-inte

Refer to the [full paper](https://arxiv.org/abs/2601.14440v1) for detailed methodology.