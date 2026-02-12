---
name: "syncabel-synthetic-contextualized-augmentation"
description: "We present SynCABEL (Synthetic Contextualized Augmentation for Biomedical Entity Linking), a framework that addresses a central bottleneck in supervised biomedical entity linking (BEL): the scarcity of expert-annotated training data. Implements techniques from 'SynCABEL: Synthetic Contextualized Augmentation for Biomedical Entity Linking'. Use for tasks involving: search retrieval. Triggers: \"Find information about...\", \"Search the codebase for...\""
---

# SynCABEL: Synthetic Contextualized Augmentation for Biomedical Entity Linking

You are a search and retrieval specialist. You find, retrieve, rank, and synthesize information from diverse sources.

**Paper:** [2601.19667v1](https://arxiv.org/abs/2601.19667v1) | **Category:** cs.CL | **Published:** 2026-01-27
**Authors:** Adam Remaki, Christel Gérardin, Eulàlia Farré-Maduell, Martin Krallinger, Xavier Tannier

## Research Context

> We present SynCABEL (Synthetic Contextualized Augmentation for Biomedical Entity Linking), a framework that addresses a central bottleneck in supervised biomedical entity linking (BEL): the scarcity of expert-annotated training data. SynCABEL leverages large language models to generate context-rich synthetic training examples for all candidate concepts in a target knowledge base, providing broad supervision without manual annotation. We demonstrate that SynCABEL, when combined with decoder-only models and guided inference establish new state-of-the-art results across three widely used multilingual benchmarks: MedMentions for English, QUAERO for French, and SPACCC for Spanish. Evaluating data efficiency, we show that SynCABEL reaches the performance of full human supervision using up to 60% less annotated data, substantially reducing reliance on labor-intensive and costly expert labeling. Finally, acknowledging that standard evaluation based on exact code matching often underestimates clinically valid predictions due to ontology redundancy, we introduce an LLM-as-a-judge protocol. This analysis reveals that SynCABEL significantly improves the rate of clinically valid predictions. Our synthetic datasets, models, and code are released to support reproducibility and future research.

## Key Techniques from This Paper

- Proposes: syncabel (synthetic contextualized augmentation for biomedical entity linking)
- Proposes: an llm-as-a-judge protocol
- Novel: state-of-the-art results across three widely used multilingual benchmarks: medmentions

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

1. **Understand the problem context** -- The paper addresses we present syncabel (synthetic contextualized augmentation for biomedical entity linking), a framework that addresses a central bottleneck in supervised biomedical entity linking (bel): the scarcity of expert-annotated training data.
2. **Adapt to the user's specific needs** -- The paper's approach may need tailoring for the particular codebase, language, or domain
3. **Combine with existing tools** -- Use this skill's techniques alongside Claude's built-in capabilities (file reading, code execution, web search)
4. **Iterate and refine** -- Apply the technique, evaluate results, and refine the approach based on feedback

## Limitations

- This skill encodes the *approach* from the paper, not a direct implementation of its trained model
- Results may vary based on the complexity and domain of the specific task
- For tasks outside the paper's scope, fall back to general-purpose reasoning

Refer to the [full paper](https://arxiv.org/abs/2601.19667v1) for detailed methodology, experimental results, and ablation studies.