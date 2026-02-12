---
name: "fine-tuning-pretrained-visionlanguage-models"
description: "Large-scale vision-language models (VLMs) such as CLIP exhibit strong zero-shot generalization, but adapting them to downstream tasks typically requires costly labeled data. Implements techniques from the paper 'Fine-tuning Pre-trained Vision-Language Models in a Human-Annotation-Free Manner' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (prompt engineering) or when the user references techniques from this research area."
---

# Fine-tuning Pre-trained Vision-Language Models in a Human-Annotation-Free Manner

**Source:** [https://arxiv.org/abs/2602.04337v1](https://arxiv.org/abs/2602.04337v1)
**Category:** cs.CV | **Published:** 2026-02-04 | **Skill Score:** 64
**Authors:** Qian-Wei Wang, Guanghao Meng, Ren Cai...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** collaborative fine-tuning (coft)
- **Leverages:** unlabeled data through a dual-model

## Workflow

1. Parse the user's information need or query
2. Formulate effective search strategies
3. Retrieve relevant documents, code, or data
4. Synthesize findings into a coherent response
5. Provide citations and references

## Research Context

> Large-scale vision-language models (VLMs) such as CLIP exhibit strong zero-shot generalization, but adapting them to downstream tasks typically requires costly labeled data. Existing unsupervised self-training methods rely on pseudo-labeling, yet often suffer from unreliable confidence filtering, confirmation bias, and underutilization of low-confidence samples. We propose Collaborative Fine-Tuning (CoFT), an unsupervised adaptation framework that leverages unlabeled data through a dual-model, c

Refer to the [full paper](https://arxiv.org/abs/2602.04337v1) for detailed methodology.