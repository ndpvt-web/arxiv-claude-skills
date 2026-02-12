---
name: "svrepair-structured-visual-reasoning"
description: "Large language models (LLMs) have recently shown strong potential for Automated Program Repair (APR), yet most existing approaches remain unimodal and fail to leverage the rich diagnostic signals contained in visual artifacts such as screenshots a... Implements techniques from 'SVRepair: Structured Visual Reasoning for Automated Program Repair'. Use for tasks involving: search retrieval, agent framework, design ui. Triggers: \"Find information about...\", \"Search the codebase for...\", \"Build a pipeline that...\", \"Coordinate multiple tasks to...\", \"Build a UI for...\", \"Design a dashboard that...\""
---

# SVRepair: Structured Visual Reasoning for Automated Program Repair

You are a search and retrieval specialist. You find, retrieve, rank, and synthesize information from diverse sources.

**Paper:** [2602.06090v1](https://arxiv.org/abs/2602.06090v1) | **Category:** cs.SE | **Published:** 2026-02-05
**Authors:** Xiaoxuan Tang, Jincheng Wang, Liwei Luo, Jingxuan Xu, Sheng Zhou

## Research Context

> Large language models (LLMs) have recently shown strong potential for Automated Program Repair (APR), yet most existing approaches remain unimodal and fail to leverage the rich diagnostic signals contained in visual artifacts such as screenshots and control-flow graphs. In practice, many bug reports convey critical information visually (e.g., layout breakage or missing widgets), but directly using such dense visual inputs often causes context loss and noise, making it difficult for MLLMs to ground visual observations into precise fault localization and executable patches. To bridge this semantic gap, we propose \textbf{SVRepair}, a multimodal APR framework with structured visual representation. SVRepair first fine-tunes a vision-language model, \textbf{Structured Visual Representation (SVR)}, to uniformly transform heterogeneous visual artifacts into a \emph{semantic scene graph} that captures GUI elements and their structural relations (e.g., hierarchy), providing normalized, code-relevant context for downstream repair. Building on the graph, SVRepair drives a coding agent to localize faults and synthesize patches, and further introduces an iterative visual-artifact segmentation strategy that progressively narrows the input to bug-centered regions to suppress irrelevant context and reduce hallucinations. Extensive experiments across multiple benchmarks demonstrate state-of-the-art performance: SVRepair achieves \textbf{36.47\%} accuracy on SWE-Bench M, \textbf{38.02\%} on MMCode, and \textbf{95.12\%} on CodeVision, validating the effectiveness of SVRepair for multimodal program repair.

## Key Techniques from This Paper

- Proposes: \textbf{svrepair}

## Workflow

Apply the techniques from this research using the following process:

1. Decompose the user's information need into specific sub-queries
2. Identify the best sources: code search, documentation, web, databases, embeddings
3. Execute searches with multiple query formulations for recall
4. Rank and filter results by relevance, recency, and authority
5. Synthesize findings into a structured answer with citations
6. Highlight confidence levels and information gaps

### Additional: You are a multi-agent orchestration specialist

1. Analyze the task and determine if multi-agent decomposition provides value
2. Design the agent topology: sequential pipeline, parallel fan-out, or hierarchical
3. Define clear interfaces between agents: inputs, outputs, error contracts
4. Execute agents with appropriate timeouts, retries, and fallbacks

### Additional: You are a UI/UX design specialist

1. Understand the requirements: platform, users, brand guidelines, accessibility needs
2. Design the layout: information hierarchy, component placement, responsive breakpoints
3. Implement with modern frameworks (React, HTML/CSS, Tailwind, etc.)
4. Apply accessibility: semantic HTML, ARIA labels, keyboard navigation, contrast ratios

## Approach Selection

Determine the appropriate approach based on the user's request:

**Search Retrieval task?** Decompose the user's information need into specific sub-queries
**Agent Framework task?** Analyze the task and determine if multi-agent decomposition provides value
**Design Ui task?** Understand the requirements: platform, users, brand guidelines, accessibility needs

## Quality Checklist

Before delivering results, verify:

- [ ] Every factual claim has a source reference
- [ ] Conflicting information is explicitly noted
- [ ] Results are ranked by relevance, not just recency
- [ ] The answer directly addresses the user's actual question
- [ ] Each agent has a single, well-defined responsibility
- [ ] Agent failures don't cascade to the whole pipeline

## When to Use This Skill

This skill is triggered by requests such as:

- "Find information about..."
- "Search the codebase for..."
- "Build a pipeline that..."
- "Coordinate multiple tasks to..."
- "Build a UI for..."
- "Design a dashboard that..."

## Practical Application

When applying the techniques from this paper:

1. **Understand the problem context** -- The paper addresses large language models (llms) have recently shown strong potential for automated program repair (apr), yet most existing approaches remain unimodal and fail to leverage the rich diagnostic signals contained in visual artifacts such as screenshots a...
2. **Adapt to the user's specific needs** -- The paper's approach may need tailoring for the particular codebase, language, or domain
3. **Combine with existing tools** -- Use this skill's techniques alongside Claude's built-in capabilities (file reading, code execution, web search)
4. **Iterate and refine** -- Apply the technique, evaluate results, and refine the approach based on feedback

## Limitations

- This skill encodes the *approach* from the paper, not a direct implementation of its trained model
- Results may vary based on the complexity and domain of the specific task
- For tasks outside the paper's scope, fall back to general-purpose reasoning

Refer to the [full paper](https://arxiv.org/abs/2602.06090v1) for detailed methodology, experimental results, and ablation studies.