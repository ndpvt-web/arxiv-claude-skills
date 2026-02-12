---
name: "endogenous-resistance-to-activation"
description: "Large language models can resist task-misaligned activation steering during inference, sometimes recovering mid-generation to produce improved responses even when steering remains active. Implements techniques from the paper 'Endogenous Resistance to Activation Steering in Language Models' for optimize prompts for better ai model performance. Use when tasks involve (prompt engineering) or when the user references techniques from this research area."
---

# Endogenous Resistance to Activation Steering in Language Models

**Source:** [https://arxiv.org/abs/2602.06941v1](https://arxiv.org/abs/2602.06941v1)
**Category:** cs.LG | **Published:** 2026-02-06 | **Skill Score:** 72
**Authors:** Alex McKenzie, Keenan Pepper, Stijn Servaes...

## Core Capability

Optimize prompts for better AI model performance.

## Workflow

1. Analyze the current prompt and its shortcomings
2. Apply prompt engineering techniques (CoT, few-shot, etc.)
3. Test prompts against diverse inputs
4. Iterate on prompt design based on results
5. Document the prompt template and its parameters

## Research Context

> Large language models can resist task-misaligned activation steering during inference, sometimes recovering mid-generation to produce improved responses even when steering remains active. We term this Endogenous Steering Resistance (ESR). Using sparse autoencoder (SAE) latents to steer model activations, we find that Llama-3.3-70B shows substantial ESR, while smaller models from the Llama-3 and Gemma-2 families exhibit the phenomenon less frequently. We identify 26 SAE latents that activate diff

Refer to the [full paper](https://arxiv.org/abs/2602.06941v1) for detailed methodology.