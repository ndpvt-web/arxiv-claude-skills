---
name: "aligning-treesearch-policies-with"
description: "Tree-search decoding is an effective form of test-time scaling for large language models (LLMs), but real-world deployment imposes a fixed per-query token budget that varies across settings. Implements techniques from the paper 'Aligning Tree-Search Policies with Fixed Token Budgets in Test-Time Scaling of LLMs' for automate deployment, ci/cd, and infrastructure tasks. Use when tasks involve (devops automation), (search & retrieval) or when the user references techniques from this research area."
---

# Aligning Tree-Search Policies with Fixed Token Budgets in Test-Time Scaling of LLMs

**Source:** [https://arxiv.org/abs/2602.09574v1](https://arxiv.org/abs/2602.09574v1)
**Category:** cs.CL | **Published:** 2026-02-10 | **Skill Score:** 58
**Authors:** Sora Miyamoto, Daisuke Oba, Naoaki Okazaki

## Core Capability

Automate deployment, CI/CD, and infrastructure tasks.

## Key Techniques

- **Proposed technique:** {budget-guided mcts} (bg-mcts)

## Workflow

1. Analyze the current infrastructure and deployment setup
2. Design automation workflows for the target environment
3. Generate configuration files (Docker, K8s, CI/CD pipelines)
4. Implement monitoring and alerting
5. Validate the automation works correctly

## Research Context

> Tree-search decoding is an effective form of test-time scaling for large language models (LLMs), but real-world deployment imposes a fixed per-query token budget that varies across settings. Existing tree-search policies are largely budget-agnostic, treating the budget as a termination condition, which can lead to late-stage over-branching or premature termination. We propose {Budget-Guided MCTS} (BG-MCTS), a tree-search decoding algorithm that aligns its search policy with the remaining token b

Refer to the [full paper](https://arxiv.org/abs/2602.09574v1) for detailed methodology.