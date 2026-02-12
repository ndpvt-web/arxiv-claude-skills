---
name: "next-concept-prediction-in"
description: "We propose Next Concept Prediction (NCP), a generative pretraining paradigm built on top of Next Token Prediction (NTP). Implements techniques from the paper 'Next Concept Prediction in Discrete Latent Space Leads to Stronger Language Models' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval) or when the user references techniques from this research area."
---

# Next Concept Prediction in Discrete Latent Space Leads to Stronger Language Models

**Source:** [https://arxiv.org/abs/2602.08984v1](https://arxiv.org/abs/2602.08984v1)
**Category:** cs.CL | **Published:** 2026-02-09 | **Skill Score:** 58
**Authors:** Yuliang Liu, Yunchong Song, Yixuan Wang...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** next concept prediction (ncp)
- **Leverages:** both ncp and ntp to drive parameter updates and generates a concept to guide the generation of the following tokens

## Workflow

1. Parse the user's information need or query
2. Formulate effective search strategies
3. Retrieve relevant documents, code, or data
4. Synthesize findings into a coherent response
5. Provide citations and references

## Research Context

> We propose Next Concept Prediction (NCP), a generative pretraining paradigm built on top of Next Token Prediction (NTP). NCP predicts discrete concepts that span multiple tokens, thereby forming a more challenging pretraining objective. Our model, ConceptLM, quantizes hidden states using Vector Quantization and constructs a concept vocabulary. It leverages both NCP and NTP to drive parameter updates and generates a concept to guide the generation of the following tokens. We train ConceptLM from 

Refer to the [full paper](https://arxiv.org/abs/2602.08984v1) for detailed methodology.