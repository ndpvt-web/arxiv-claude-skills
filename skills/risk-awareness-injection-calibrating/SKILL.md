---
name: "risk-awareness-injection-calibrating"
description: "Vision language models (VLMs) extend the reasoning capabilities of large language models (LLMs) to cross-modal settings, yet remain highly vulnerable to multimodal jailbreak attacks. Implements techniques from the paper 'Risk Awareness Injection: Calibrating Vision-Language Models for Safety without Compromising Utility' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# Risk Awareness Injection: Calibrating Vision-Language Models for Safety without Compromising Utility

**Source:** [https://arxiv.org/abs/2602.03402v2](https://arxiv.org/abs/2602.03402v2)
**Category:** cs.AI | **Published:** 2026-02-03 | **Skill Score:** 69
**Authors:** Mengxuan Wang, Yuxin Chen, Gang Xu...

## Core Capability

Search, retrieve, and synthesize information.

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

> Vision language models (VLMs) extend the reasoning capabilities of large language models (LLMs) to cross-modal settings, yet remain highly vulnerable to multimodal jailbreak attacks. Existing defenses predominantly rely on safety fine-tuning or aggressive token manipulations, incurring substantial training costs or significantly degrading utility. Recent research shows that LLMs inherently recognize unsafe content in text, and the incorporation of visual inputs in VLMs frequently dilutes risk-re

Refer to the [full paper](https://arxiv.org/abs/2602.03402v2) for detailed methodology.