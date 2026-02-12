---
name: "noisy-but-valid-robust"
description: "Reliable certification of Large Language Models (LLMs)-verifying that failure rates are below a safety threshold-is critical yet challenging. Implements techniques from the paper 'Noisy but Valid: Robust Statistical Evaluation of LLMs with Imperfect Judges' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval) or when the user references techniques from this research area."
---

# Noisy but Valid: Robust Statistical Evaluation of LLMs with Imperfect Judges

**Source:** [https://arxiv.org/abs/2601.20913v1](https://arxiv.org/abs/2601.20913v1)
**Category:** cs.LG | **Published:** 2026-01-28 | **Skill Score:** 85
**Authors:** Chen Feng, Minghe Shen, Ananth Balashankar...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** a "noisy but valid" hypothesis testing framework to address this
- **Leverages:** a small human-labelled calibration set to estimate the judge's true positive and false positive rates (tpr/fpr)

## Workflow

1. Parse the user's information need or query
2. Formulate effective search strategies
3. Retrieve relevant documents, code, or data
4. Synthesize findings into a coherent response
5. Provide citations and references

## Research Context

> Reliable certification of Large Language Models (LLMs)-verifying that failure rates are below a safety threshold-is critical yet challenging. While "LLM-as-a-Judge" offers scalability, judge imperfections, noise, and bias can invalidate statistical guarantees. We introduce a "Noisy but Valid" hypothesis testing framework to address this. By leveraging a small human-labelled calibration set to estimate the judge's True Positive and False Positive Rates (TPR/FPR), we derive a variance-corrected cr

Refer to the [full paper](https://arxiv.org/abs/2601.20913v1) for detailed methodology.