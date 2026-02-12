---
name: "iris-implicit-rewardguided-internal"
description: "Hallucination remains a fundamental challenge for Multimodal Large Language Models (MLLMs). Implements techniques from the paper 'IRIS: Implicit Reward-Guided Internal Sifting for Mitigating Multimodal Hallucination' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval) or when the user references techniques from this research area."
---

# IRIS: Implicit Reward-Guided Internal Sifting for Mitigating Multimodal Hallucination

**Source:** [https://arxiv.org/abs/2602.01769v2](https://arxiv.org/abs/2602.01769v2)
**Category:** cs.LG | **Published:** 2026-02-02 | **Skill Score:** 62
**Authors:** Yuanshuai Li, Yuping Yan, Jirui Han...

## Core Capability

Search, retrieve, and synthesize information.

## Workflow

1. Parse the user's information need or query
2. Formulate effective search strategies
3. Retrieve relevant documents, code, or data
4. Synthesize findings into a coherent response
5. Provide citations and references

## Research Context

> Hallucination remains a fundamental challenge for Multimodal Large Language Models (MLLMs). While Direct Preference Optimization (DPO) is a key alignment framework, existing approaches often rely heavily on costly external evaluators for scoring or rewriting, incurring off-policy learnability gaps and discretization loss. Due to the lack of access to internal states, such feedback overlooks the fine-grained conflicts between different modalities that lead to hallucinations during generation.   T

Refer to the [full paper](https://arxiv.org/abs/2602.01769v2) for detailed methodology.