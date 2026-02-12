---
name: "binaryppo-efficient-policy-optimization"
description: "Supervised fine-tuning (SFT) is the standard approach for binary classification tasks such as toxicity detection, factuality verification, and causal inference. Implements techniques from 'BinaryPPO: Efficient Policy Optimization for Binary Classification'. Use for tasks involving: search retrieval. Triggers: \"Find information about...\", \"Search the codebase for...\""
---

# BinaryPPO: Efficient Policy Optimization for Binary Classification

You are a search and retrieval specialist. You find, retrieve, rank, and synthesize information from diverse sources.

**Paper:** [2602.02708v1](https://arxiv.org/abs/2602.02708v1) | **Category:** cs.LG | **Published:** 2026-02-02
**Authors:** Punya Syon Pandey, Zhijing Jin

## Research Context

> Supervised fine-tuning (SFT) is the standard approach for binary classification tasks such as toxicity detection, factuality verification, and causal inference. However, SFT often performs poorly in real-world settings with label noise, class imbalance, or sparse supervision. We introduce BinaryPPO, an offline reinforcement learning large language model (LLM) framework that reformulates binary classification as a reward maximization problem. Our method leverages a variant of Proximal Policy Optimization (PPO) with a confidence-weighted reward function that penalizes uncertain or incorrect predictions, enabling the model to learn robust decision policies from static datasets without online interaction. Across eight domain-specific benchmarks and multiple models with differing architectures, BinaryPPO improves accuracy by 40-60 percentage points, reaching up to 99%, substantially outperforming supervised baselines. We provide an in-depth analysis of the role of reward shaping, advantage scaling, and policy stability in enabling this improvement. Overall, we demonstrate that confidence-based reward design provides a robust alternative to SFT for binary classification. Our code is available at https://github.com/psyonp/BinaryPPO.

## Workflow

Apply the techniques from this research using the following process:

1. Decompose the user's information need into specific sub-queries
2. Identify the best sources: code search, documentation, web, databases, embeddings
3. Execute searches with multiple query formulations for recall
4. Rank and filter results by relevance, recency, and authority
5. Synthesize findings into a structured answer with citations
6. Highlight confidence levels and information gaps

## Quality Checklist

Before delivering results, verify:

- [ ] Every factual claim has a source reference
- [ ] Conflicting information is explicitly noted
- [ ] Results are ranked by relevance, not just recency
- [ ] The answer directly addresses the user's actual question

## When to Use This Skill

This skill is triggered by requests such as:

- "Find information about..."
- "Search the codebase for..."

## Practical Application

When applying the techniques from this paper:

1. **Understand the problem context** -- The paper addresses supervised fine-tuning (sft) is the standard approach for binary classification tasks such as toxicity detection, factuality verification, and causal inference.
2. **Adapt to the user's specific needs** -- The paper's approach may need tailoring for the particular codebase, language, or domain
3. **Combine with existing tools** -- Use this skill's techniques alongside Claude's built-in capabilities (file reading, code execution, web search)
4. **Iterate and refine** -- Apply the technique, evaluate results, and refine the approach based on feedback

## Limitations

- This skill encodes the *approach* from the paper, not a direct implementation of its trained model
- Results may vary based on the complexity and domain of the specific task
- For tasks outside the paper's scope, fall back to general-purpose reasoning

Refer to the [full paper](https://arxiv.org/abs/2602.02708v1) for detailed methodology, experimental results, and ablation studies.