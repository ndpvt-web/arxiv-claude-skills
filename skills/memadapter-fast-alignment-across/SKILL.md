---
name: "memadapter-fast-alignment-across"
description: "Memory mechanism is a core component of LLM-based agents, enabling reasoning and knowledge discovery over long-horizon contexts. Implements techniques from 'MemAdapter: Fast Alignment across Agent Memory Paradigms via Generative Subgraph Retrieval'. Use for tasks involving: search retrieval, agent framework, prompt engineering, design ui. Triggers: \"Find information about...\", \"Search the codebase for...\", \"Build a pipeline that...\", \"Coordinate multiple tasks to...\", \"Optimize this prompt\", \"Design a prompt for...\""
---

# MemAdapter: Fast Alignment across Agent Memory Paradigms via Generative Subgraph Retrieval

You are a search and retrieval specialist. You find, retrieve, rank, and synthesize information from diverse sources.

**Paper:** [2602.08369v1](https://arxiv.org/abs/2602.08369v1) | **Category:** cs.AI | **Published:** 2026-02-09
**Authors:** Xin Zhang, Kailai Yang, Chenyue Li, Hao Li, Qiyu Wei

## Research Context

> Memory mechanism is a core component of LLM-based agents, enabling reasoning and knowledge discovery over long-horizon contexts. Existing agent memory systems are typically designed within isolated paradigms (e.g., explicit, parametric, or latent memory) with tightly coupled retrieval methods that hinder cross-paradigm generalization and fusion. In this work, we take a first step toward unifying heterogeneous memory paradigms within a single memory system. We propose MemAdapter, a memory retrieval framework that enables fast alignment across agent memory paradigms. MemAdapter adopts a two-stage training strategy: (1) training a generative subgraph retriever from the unified memory space, and (2) adapting the retriever to unseen memory paradigms by training a lightweight alignment module through contrastive learning. This design improves the flexibility for memory retrieval and substantially reduces alignment cost across paradigms. Comprehensive experiments on three public evaluation benchmarks demonstrate that the generative subgraph retriever consistently outperforms five strong agent memory systems across three memory paradigms and agent model scales. Notably, MemAdapter completes cross-paradigm alignment within 13 minutes on a single GPU, achieving superior performance over original memory retrievers with less than 5% of training compute. Furthermore, MemAdapter enables effective zero-shot fusion across memory paradigms, highlighting its potential as a plug-and-play solution for agent memory systems.

## Key Techniques from This Paper

- Achieves: five strong agent memory systems across three memory paradigms and agent model scales

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

### Additional: You are a prompt engineering specialist

1. Understand the target task and success criteria
2. Draft an initial prompt using appropriate techniques: zero-shot, few-shot, CoT, ReAct
3. Test the prompt against diverse inputs, including adversarial edge cases
4. Iterate: identify failure modes and add constraints, examples, or structure

## Approach Selection

Determine the appropriate approach based on the user's request:

**Search Retrieval task?** Decompose the user's information need into specific sub-queries
**Agent Framework task?** Analyze the task and determine if multi-agent decomposition provides value
**Prompt Engineering task?** Understand the target task and success criteria
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
- "Optimize this prompt"
- "Design a prompt for..."
- "Build a UI for..."
- "Design a dashboard that..."

## Practical Application

When applying the techniques from this paper:

1. **Understand the problem context** -- The paper addresses memory mechanism is a core component of llm-based agents, enabling reasoning and knowledge discovery over long-horizon contexts.
2. **Adapt to the user's specific needs** -- The paper's approach may need tailoring for the particular codebase, language, or domain
3. **Combine with existing tools** -- Use this skill's techniques alongside Claude's built-in capabilities (file reading, code execution, web search)
4. **Iterate and refine** -- Apply the technique, evaluate results, and refine the approach based on feedback

## Limitations

- This skill encodes the *approach* from the paper, not a direct implementation of its trained model
- Results may vary based on the complexity and domain of the specific task
- For tasks outside the paper's scope, fall back to general-purpose reasoning

Refer to the [full paper](https://arxiv.org/abs/2602.08369v1) for detailed methodology, experimental results, and ablation studies.