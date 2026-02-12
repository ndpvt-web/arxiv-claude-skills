---
name: "mrad-zeroshot-anomaly-detection"
description: "Zero-shot anomaly detection (ZSAD) often leverages pretrained vision or vision-language models, but many existing methods use prompt learning or complex modeling to fit the data distribution, resul... Implements techniques from the paper 'MRAD: Zero-Shot Anomaly Detection with Memory-Driven Retrieval' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (prompt engineering) or when the user references techniques from this research area."
---

# MRAD: Zero-Shot Anomaly Detection with Memory-Driven Retrieval

**Source:** [https://arxiv.org/abs/2602.00522v1](https://arxiv.org/abs/2602.00522v1)
**Category:** cs.CV | **Published:** 2026-01-31 | **Skill Score:** 62
**Authors:** Chaoran Xu, Chengkan Lv, Qiyu Chen...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** memory-retrieval anomaly detection method (mrad)
- **Leverages:** pretrained vision or vision-language models
- **Retrieval-augmented** approach for grounding responses in external knowledge

## Workflow

1. Parse the user's information need or query
2. Formulate effective search strategies
3. Retrieve relevant documents, code, or data
4. Synthesize findings into a coherent response
5. Provide citations and references

## Research Context

> Zero-shot anomaly detection (ZSAD) often leverages pretrained vision or vision-language models, but many existing methods use prompt learning or complex modeling to fit the data distribution, resulting in high training or inference cost and limited cross-domain stability. To address these limitations, we propose Memory-Retrieval Anomaly Detection method (MRAD), a unified framework that replaces parametric fitting with a direct memory retrieval. The train-free base model, MRAD-TF, freezes the CLI

Refer to the [full paper](https://arxiv.org/abs/2602.00522v1) for detailed methodology.