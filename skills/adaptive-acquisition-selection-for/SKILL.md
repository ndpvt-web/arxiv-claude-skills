---
name: "adaptive-acquisition-selection-for"
description: "Bayesian Optimization critically depends on the choice of acquisition function, but no single strategy is universally optimal; the best choice is non-stationary and problem-dependent. Implements techniques from the paper 'Adaptive Acquisition Selection for Bayesian Optimization with Large Language Models' for optimize prompts for better ai model performance. Use when tasks involve (prompt engineering) or when the user references techniques from this research area."
---

# Adaptive Acquisition Selection for Bayesian Optimization with Large Language Models

**Source:** [https://arxiv.org/abs/2602.07904v1](https://arxiv.org/abs/2602.07904v1)
**Category:** cs.LG | **Published:** 2026-02-08 | **Skill Score:** 61
**Authors:** Giang Ngo, Dat Phan Trong, Dang Nguyen...

## Core Capability

Optimize prompts for better AI model performance.

## Key Techniques

- **Novel approach:** framework that casts a pre-trained large language model

## Workflow

1. Analyze the current prompt and its shortcomings
2. Apply prompt engineering techniques (CoT, few-shot, etc.)
3. Test prompts against diverse inputs
4. Iterate on prompt design based on results
5. Document the prompt template and its parameters

## Research Context

> Bayesian Optimization critically depends on the choice of acquisition function, but no single strategy is universally optimal; the best choice is non-stationary and problem-dependent. Existing adaptive portfolio methods often base their decisions on past function values while ignoring richer information like remaining budget or surrogate model characteristics. To address this, we introduce LMABO, a novel framework that casts a pre-trained Large Language Model (LLM) as a zero-shot, online strateg

Refer to the [full paper](https://arxiv.org/abs/2602.07904v1) for detailed methodology.