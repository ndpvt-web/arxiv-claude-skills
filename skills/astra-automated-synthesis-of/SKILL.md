---
name: "astra-automated-synthesis-of"
description: "Large language models (LLMs) are increasingly used as tool-augmented agents for multi-step decision making, yet training robust tool-using agents remains challenging. Implements techniques from 'ASTRA: Automated Synthesis of agentic Trajectories and Reinforcement Arenas'. Use for tasks involving: search retrieval, agent framework, design ui. Triggers: \"Find information about...\", \"Search the codebase for...\", \"Build a pipeline that...\", \"Coordinate multiple tasks to...\", \"Build a UI for...\", \"Design a dashboard that...\""
---

# ASTRA: Automated Synthesis of agentic Trajectories and Reinforcement Arenas

You are a search and retrieval specialist. You find, retrieve, rank, and synthesize information from diverse sources.

**Paper:** [2601.21558v2](https://arxiv.org/abs/2601.21558v2) | **Category:** cs.CL | **Published:** 2026-01-29
**Authors:** Xiaoyu Tian, Haotian Wang, Shuaiting Chen, Hao Zhou, Kaichi Yu

## Research Context

> Large language models (LLMs) are increasingly used as tool-augmented agents for multi-step decision making, yet training robust tool-using agents remains challenging. Existing methods still require manual intervention, depend on non-verifiable simulated environments, rely exclusively on either supervised fine-tuning (SFT) or reinforcement learning (RL), and struggle with stable long-horizon, multi-turn learning. To address these challenges, we introduce ASTRA, a fully automated end-to-end framework for training tool-augmented language model agents via scalable data synthesis and verifiable reinforcement learning. ASTRA integrates two complementary components. First, a pipeline that leverages the static topology of tool-call graphs synthesizes diverse, structurally grounded trajectories, instilling broad and transferable tool-use competence. Second, an environment synthesis framework that captures the rich, compositional topology of human semantic reasoning converts decomposed question-answer traces into independent, code-executable, and rule-verifiable environments, enabling deterministic multi-turn RL. Based on this method, we develop a unified training methodology that integrates SFT with online RL using trajectory-level rewards to balance task completion and interaction efficiency. Experiments on multiple agentic tool-use benchmarks demonstrate that ASTRA-trained models achieve state-of-the-art performance at comparable scales, approaching closed-source systems while preserving core reasoning ability. We release the full pipelines, environments, and trained models at https://github.com/LianjiaTech/astra.

## Key Techniques from This Paper

- Proposes: a unified training methodology that integrates sft with online rl using trajectory-level rewards to balance task completion and interaction efficiency
- Achieves: state-of-the-art performance at comparable scales

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

1. **Understand the problem context** -- The paper addresses large language models (llms) are increasingly used as tool-augmented agents for multi-step decision making, yet training robust tool-using agents remains challenging.
2. **Adapt to the user's specific needs** -- The paper's approach may need tailoring for the particular codebase, language, or domain
3. **Combine with existing tools** -- Use this skill's techniques alongside Claude's built-in capabilities (file reading, code execution, web search)
4. **Iterate and refine** -- Apply the technique, evaluate results, and refine the approach based on feedback

## Limitations

- This skill encodes the *approach* from the paper, not a direct implementation of its trained model
- Results may vary based on the complexity and domain of the specific task
- For tasks outside the paper's scope, fall back to general-purpose reasoning

Refer to the [full paper](https://arxiv.org/abs/2601.21558v2) for detailed methodology, experimental results, and ablation studies.