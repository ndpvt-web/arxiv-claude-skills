---
name: "control-reinforcement-learning-tokenlevel"
description: "Sparse autoencoders (SAEs) decompose language model activations into interpretable features, but existing methods reveal only which features activate, not which change model outputs when amplified. Implements techniques from the paper 'Control Reinforcement Learning: Token-Level Mechanistic Analysis via Learned SAE Feature Steering' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval) or when the user references techniques from this research area."
---

# Control Reinforcement Learning: Token-Level Mechanistic Analysis via Learned SAE Feature Steering

**Source:** [https://arxiv.org/abs/2602.10437v1](https://arxiv.org/abs/2602.10437v1)
**Category:** cs.LG | **Published:** 2026-02-11 | **Skill Score:** 58
**Authors:** Seonglae Cho, Zekun Wu, Adriano Koshiyama

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** control reinforcement learning (crl)

## Workflow

1. Parse the user's information need or query
2. Formulate effective search strategies
3. Retrieve relevant documents, code, or data
4. Synthesize findings into a coherent response
5. Provide citations and references

## Research Context

> Sparse autoencoders (SAEs) decompose language model activations into interpretable features, but existing methods reveal only which features activate, not which change model outputs when amplified. We introduce Control Reinforcement Learning (CRL), which trains a policy to select SAE features for steering at each token, producing interpretable intervention logs: the learned policy identifies features that change model outputs when amplified. Adaptive Feature Masking encourages diverse feature di

Refer to the [full paper](https://arxiv.org/abs/2602.10437v1) for detailed methodology.