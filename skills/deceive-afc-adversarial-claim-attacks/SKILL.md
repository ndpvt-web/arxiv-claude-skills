---
name: "deceive-afc-adversarial-claim-attacks"
description: "Fact-checking systems with search-enabled large language models (LLMs) have shown strong potential for verifying claims by dynamically retrieving external evidence. Implements techniques from the paper 'DECEIVE-AFC: Adversarial Claim Attacks against Search-Enabled LLM-based Fact-Checking Systems' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# DECEIVE-AFC: Adversarial Claim Attacks against Search-Enabled LLM-based Fact-Checking Systems

**Source:** [https://arxiv.org/abs/2602.02569v1](https://arxiv.org/abs/2602.02569v1)
**Category:** cs.CR | **Published:** 2026-01-31 | **Skill Score:** 65
**Authors:** Haoran Ou, Kangjie Chen, Gelei Deng...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** deceive-afc

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

> Fact-checking systems with search-enabled large language models (LLMs) have shown strong potential for verifying claims by dynamically retrieving external evidence. However, the robustness of such systems against adversarial attack remains insufficiently understood. In this work, we study adversarial claim attacks against search-enabled LLM-based fact-checking systems under a realistic input-only threat model. We propose DECEIVE-AFC, an agent-based adversarial attack framework that integrates no

Refer to the [full paper](https://arxiv.org/abs/2602.02569v1) for detailed methodology.