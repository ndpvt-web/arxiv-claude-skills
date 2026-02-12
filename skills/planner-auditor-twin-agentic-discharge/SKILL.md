---
name: "planner-auditor-twin-agentic-discharge"
description: "Objective: Large language models (LLMs) show promise for clinical discharge planning, but their use is constrained by hallucination, omissions, and miscalibrated confidence. Implements techniques from the paper 'Planner-Auditor Twin: Agentic Discharge Planning with FHIR-Based LLM Planning, Guideline Recall, Optional Caching and Self-Improvement' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# Planner-Auditor Twin: Agentic Discharge Planning with FHIR-Based LLM Planning, Guideline Recall, Optional Caching and Self-Improvement

**Source:** [https://arxiv.org/abs/2601.21113v1](https://arxiv.org/abs/2601.21113v1)
**Category:** cs.AI | **Published:** 2026-01-28 | **Skill Score:** 75
**Authors:** Kaiyuan Wu, Aditya Nagori, Rishikesan Kamaleswaran

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** a self-improving

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

> Objective: Large language models (LLMs) show promise for clinical discharge planning, but their use is constrained by hallucination, omissions, and miscalibrated confidence. We introduce a self-improving, cache-optional Planner-Auditor framework that improves safety and reliability by decoupling generation from deterministic validation and targeted replay.   Materials and Methods: We implemented an agentic, retrospective, FHIR-native evaluation pipeline using MIMIC-IV-on-FHIR. For each patient, 

Refer to the [full paper](https://arxiv.org/abs/2601.21113v1) for detailed methodology.