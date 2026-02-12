---
name: "noisy-but-valid-robust"
description: "Reliable certification of Large Language Models (LLMs)-verifying that failure rates are below a safety threshold-is critical yet challenging. Implements techniques from 'Noisy but Valid: Robust Statistical Evaluation of LLMs with Imperfect Judges'. Use for tasks involving: search retrieval. Triggers: \"Find information about...\", \"Search the codebase for...\""
---

# Noisy but Valid: Robust Statistical Evaluation of LLMs with Imperfect Judges

You are a search and retrieval specialist. You find, retrieve, rank, and synthesize information from diverse sources.

**Paper:** [2601.20913v1](https://arxiv.org/abs/2601.20913v1) | **Category:** cs.LG | **Published:** 2026-01-28
**Authors:** Chen Feng, Minghe Shen, Ananth Balashankar, Carsten Gerner-Beuerle, Miguel R. D. Rodrigues

## Research Context

> Reliable certification of Large Language Models (LLMs)-verifying that failure rates are below a safety threshold-is critical yet challenging. While "LLM-as-a-Judge" offers scalability, judge imperfections, noise, and bias can invalidate statistical guarantees. We introduce a "Noisy but Valid" hypothesis testing framework to address this. By leveraging a small human-labelled calibration set to estimate the judge's True Positive and False Positive Rates (TPR/FPR), we derive a variance-corrected critical threshold applied to a large judge-labelled dataset. Crucially, our framework theoretically guarantees finite-sample Type-I error control (validity) despite calibration uncertainty. This distinguishes our work from Prediction-Powered Inference (PPI), positioning our method as a diagnostic tool that explicitly models judge behavior rather than a black-box estimator. Our contributions include: (1) Theoretical Guarantees: We derive the exact conditions under which noisy testing yields higher statistical power than direct evaluation; (2) Empirical Validation: Experiments on Jigsaw Comment, Hate Speech and SafeRLHF confirm our theory; (3) The Oracle Gap: We reveal a significant performance gap between practical methods and the theoretical "Oracle" (perfectly known judge parameters), quantifying the cost of estimation. Specifically, we provide the first systematic treatment of the imperfect-judge setting, yielding interpretable diagnostics of judge reliability and clarifying how evaluation power depends on judge quality, dataset size, and certification levels. Together, these results sharpen understanding of statistical evaluation with LLM judges, and highlight trade-offs among competing inferential tools.

## Key Techniques from This Paper

- Proposes: a "noisy but valid" hypothesis testing framework to address this

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

1. **Understand the problem context** -- The paper addresses reliable certification of large language models (llms)-verifying that failure rates are below a safety threshold-is critical yet challenging.
2. **Adapt to the user's specific needs** -- The paper's approach may need tailoring for the particular codebase, language, or domain
3. **Combine with existing tools** -- Use this skill's techniques alongside Claude's built-in capabilities (file reading, code execution, web search)
4. **Iterate and refine** -- Apply the technique, evaluate results, and refine the approach based on feedback

## Limitations

- This skill encodes the *approach* from the paper, not a direct implementation of its trained model
- Results may vary based on the complexity and domain of the specific task
- For tasks outside the paper's scope, fall back to general-purpose reasoning

Refer to the [full paper](https://arxiv.org/abs/2601.20913v1) for detailed methodology, experimental results, and ablation studies.