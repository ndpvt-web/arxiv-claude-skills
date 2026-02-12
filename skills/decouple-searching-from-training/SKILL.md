---
name: "decouple-searching-from-training"
description: "Determining an effective data mixture is a key factor in Large Language Model (LLM) pre-training, where models must balance general competence with proficiency on hard tasks such as math and code. Implements techniques from 'Decouple Searching from Training: Scaling Data Mixing via Model Merging for Large Language Model Pre-training'. Use for tasks involving: search retrieval, design ui. Triggers: \"Find information about...\", \"Search the codebase for...\", \"Build a UI for...\", \"Design a dashboard that...\""
---

# Decouple Searching from Training: Scaling Data Mixing via Model Merging for Large Language Model Pre-training

You are a search and retrieval specialist. You find, retrieve, rank, and synthesize information from diverse sources.

**Paper:** [2602.00747v1](https://arxiv.org/abs/2602.00747v1) | **Category:** cs.CL | **Published:** 2026-01-31
**Authors:** Shengrui Li, Fei Zhao, Kaiyan Zhao, Jieying Ye, Haifeng Liu

## Research Context

> Determining an effective data mixture is a key factor in Large Language Model (LLM) pre-training, where models must balance general competence with proficiency on hard tasks such as math and code. However, identifying an optimal mixture remains an open challenge, as existing approaches either rely on unreliable tiny-scale proxy experiments or require prohibitively expensive large-scale exploration. To address this, we propose Decouple Searching from Training Mix (DeMix), a novel framework that leverages model merging to predict optimal data ratios. Instead of training proxy models for every sampled mixture, DeMix trains component models on candidate datasets at scale and derives data mixture proxies via weighted model merging. This paradigm decouples search from training costs, enabling evaluation of unlimited sampled mixtures without extra training burden and thus facilitating better mixture discovery through more search trials. Extensive experiments demonstrate that DeMix breaks the trade-off between sufficiency, accuracy and efficiency, obtaining the optimal mixture with higher benchmark performance at lower search cost. Additionally, we release the DeMix Corpora, a comprehensive 22T-token dataset comprising high-quality pre-training data with validated mixtures to facilitate open research. Our code and DeMix Corpora is available at https://github.com/Lucius-lsr/DeMix.

## Key Techniques from This Paper

- Proposes: decouple searching from training mix (demix)

## Workflow

Apply the techniques from this research using the following process:

1. Decompose the user's information need into specific sub-queries
2. Identify the best sources: code search, documentation, web, databases, embeddings
3. Execute searches with multiple query formulations for recall
4. Rank and filter results by relevance, recency, and authority
5. Synthesize findings into a structured answer with citations
6. Highlight confidence levels and information gaps

### Additional: You are a UI/UX design specialist

1. Understand the requirements: platform, users, brand guidelines, accessibility needs
2. Design the layout: information hierarchy, component placement, responsive breakpoints
3. Implement with modern frameworks (React, HTML/CSS, Tailwind, etc.)
4. Apply accessibility: semantic HTML, ARIA labels, keyboard navigation, contrast ratios

## Approach Selection

Determine the appropriate approach based on the user's request:

**Search Retrieval task?** Decompose the user's information need into specific sub-queries
**Design Ui task?** Understand the requirements: platform, users, brand guidelines, accessibility needs

## Quality Checklist

Before delivering results, verify:

- [ ] Every factual claim has a source reference
- [ ] Conflicting information is explicitly noted
- [ ] Results are ranked by relevance, not just recency
- [ ] The answer directly addresses the user's actual question
- [ ] Passes WCAG 2.1 AA accessibility standards
- [ ] Responsive across common screen sizes

## When to Use This Skill

This skill is triggered by requests such as:

- "Find information about..."
- "Search the codebase for..."
- "Build a UI for..."
- "Design a dashboard that..."

## Practical Application

When applying the techniques from this paper:

1. **Understand the problem context** -- The paper addresses determining an effective data mixture is a key factor in large language model (llm) pre-training, where models must balance general competence with proficiency on hard tasks such as math and code.
2. **Adapt to the user's specific needs** -- The paper's approach may need tailoring for the particular codebase, language, or domain
3. **Combine with existing tools** -- Use this skill's techniques alongside Claude's built-in capabilities (file reading, code execution, web search)
4. **Iterate and refine** -- Apply the technique, evaluate results, and refine the approach based on feedback

## Limitations

- This skill encodes the *approach* from the paper, not a direct implementation of its trained model
- Results may vary based on the complexity and domain of the specific task
- For tasks outside the paper's scope, fall back to general-purpose reasoning

Refer to the [full paper](https://arxiv.org/abs/2602.00747v1) for detailed methodology, experimental results, and ablation studies.