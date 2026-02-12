---
name: "tooltok-tool-tokenization-for"
description: "Existing GUI agent models relying on coordinate-based one-step visual grounding struggle with generalizing to varying input resolutions and aspect ratios. Implements techniques from 'ToolTok: Tool Tokenization for Efficient and Generalizable GUI Agents'. Use for tasks involving: search retrieval, agent framework. Triggers: \"Find information about...\", \"Search the codebase for...\", \"Build a pipeline that...\", \"Coordinate multiple tasks to...\""
---

# ToolTok: Tool Tokenization for Efficient and Generalizable GUI Agents

You are a search and retrieval specialist. You find, retrieve, rank, and synthesize information from diverse sources.

**Paper:** [2602.02548v1](https://arxiv.org/abs/2602.02548v1) | **Category:** cs.LG | **Published:** 2026-01-30
**Authors:** Xiaoce Wang, Guibin Zhang, Junzhe Li, Jinzhe Tu, Chun Li

## Research Context

> Existing GUI agent models relying on coordinate-based one-step visual grounding struggle with generalizing to varying input resolutions and aspect ratios. Alternatives introduce coordinate-free strategies yet suffer from learning under severe data scarcity. To address the limitations, we propose ToolTok, a novel paradigm of multi-step pathfinding for GUI agents, where operations are modeled as a sequence of progressive tool usage. Specifically, we devise tools aligned with human interaction habits and represent each tool using learnable token embeddings. To enable efficient embedding learning under limited supervision, ToolTok introduces a semantic anchoring mechanism that grounds each tool with semantically related concepts as natural inductive bias. To further enable a pre-trained large language model to progressively acquire tool semantics, we construct an easy-to-hard curriculum consisting of three tasks: token definition question-answering, pure text-guided tool selection, and simplified visual pathfinding. Extensive experiments on multiple benchmarks show that ToolTok achieves superior performance among models of comparable scale (4B) and remains competitive with a substantially larger model (235B). Notably, these results are obtained using less than 1% of the training data required by other post-training approaches. In addition, ToolTok demonstrates strong generalization across unseen scenarios. Our training & inference code is open-source at https://github.com/ZephinueCode/ToolTok.

## Key Techniques from This Paper

- Novel: paradigm of multi-step pathfinding
- Achieves: superior performance among models of comparable scale (4b) and remains competitive with a substantially larger model (235b)

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

## Approach Selection

Determine the appropriate approach based on the user's request:

**Search Retrieval task?** Decompose the user's information need into specific sub-queries
**Agent Framework task?** Analyze the task and determine if multi-agent decomposition provides value

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

## Practical Application

When applying the techniques from this paper:

1. **Understand the problem context** -- The paper addresses existing gui agent models relying on coordinate-based one-step visual grounding struggle with generalizing to varying input resolutions and aspect ratios.
2. **Adapt to the user's specific needs** -- The paper's approach may need tailoring for the particular codebase, language, or domain
3. **Combine with existing tools** -- Use this skill's techniques alongside Claude's built-in capabilities (file reading, code execution, web search)
4. **Iterate and refine** -- Apply the technique, evaluate results, and refine the approach based on feedback

## Limitations

- This skill encodes the *approach* from the paper, not a direct implementation of its trained model
- Results may vary based on the complexity and domain of the specific task
- For tasks outside the paper's scope, fall back to general-purpose reasoning

Refer to the [full paper](https://arxiv.org/abs/2602.02548v1) for detailed methodology, experimental results, and ablation studies.