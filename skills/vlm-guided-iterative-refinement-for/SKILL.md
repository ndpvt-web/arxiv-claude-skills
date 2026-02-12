---
name: "vlm-guided-iterative-refinement-for"
description: "Surgical image segmentation is essential for robot-assisted surgery and intraoperative guidance. Implements techniques from the paper 'VLM-Guided Iterative Refinement for Surgical Image Segmentation with Foundation Models' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# VLM-Guided Iterative Refinement for Surgical Image Segmentation with Foundation Models

**Source:** [https://arxiv.org/abs/2602.09252v1](https://arxiv.org/abs/2602.09252v1)
**Category:** cs.CV | **Published:** 2026-02-09 | **Skill Score:** 76
**Authors:** Ange Lou, Yamin Li, Qi Chang...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Leverages:** a fine-tuned sam3 for initial segmentation

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

> Surgical image segmentation is essential for robot-assisted surgery and intraoperative guidance. However, existing methods are constrained to predefined categories, produce one-shot predictions without adaptive refinement, and lack mechanisms for clinician interaction. We propose IR-SIS, an iterative refinement system for surgical image segmentation that accepts natural language descriptions. IR-SIS leverages a fine-tuned SAM3 for initial segmentation, employs a Vision-Language Model to detect i

Refer to the [full paper](https://arxiv.org/abs/2602.09252v1) for detailed methodology.