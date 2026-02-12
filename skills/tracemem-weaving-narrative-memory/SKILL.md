---
name: "tracemem-weaving-narrative-memory"
description: "Sustaining long-term interactions remains a bottleneck for Large Language Models (LLMs), as their limited context windows struggle to manage dialogue histories that extend over time. Implements techniques from 'TraceMem: Weaving Narrative Memory Schemata from User Conversational Traces'. Use for tasks involving: search retrieval, agent framework, database query. Triggers: \"Find information about...\", \"Search the codebase for...\", \"Build a pipeline that...\", \"Coordinate multiple tasks to...\", \"Write a SQL query to...\", \"How do I query for...\""
---

# TraceMem: Weaving Narrative Memory Schemata from User Conversational Traces

You are a search and retrieval specialist. You find, retrieve, rank, and synthesize information from diverse sources.

**Paper:** [2602.09712v1](https://arxiv.org/abs/2602.09712v1) | **Category:** cs.CL | **Published:** 2026-02-10
**Authors:** Yiming Shu, Pei Liu, Tiange Zhang, Ruiyang Gao, Jun Ma

## Research Context

> Sustaining long-term interactions remains a bottleneck for Large Language Models (LLMs), as their limited context windows struggle to manage dialogue histories that extend over time. Existing memory systems often treat interactions as disjointed snippets, failing to capture the underlying narrative coherence of the dialogue stream. We propose TraceMem, a cognitively-inspired framework that weaves structured, narrative memory schemata from user conversational traces through a three-stage pipeline: (1) Short-term Memory Processing, which employs a deductive topic segmentation approach to demarcate episode boundaries and extract semantic representation; (2) Synaptic Memory Consolidation, a process that summarizes episodes into episodic memories before distilling them alongside semantics into user-specific traces; and (3) Systems Memory Consolidation, which utilizes two-stage hierarchical clustering to organize these traces into coherent, time-evolving narrative threads under unifying themes. These threads are encapsulated into structured user memory cards, forming narrative memory schemata. For memory utilization, we provide an agentic search mechanism to enhance reasoning process. Evaluation on the LoCoMo benchmark shows that TraceMem achieves state-of-the-art performance with a brain-inspired architecture. Analysis shows that by constructing coherent narratives, it surpasses baselines in multi-hop and temporal reasoning, underscoring its essential role in deep narrative comprehension. Additionally, we provide an open discussion on memory systems, offering our perspectives and future outlook on the field. Our code implementation is available at: https://github.com/YimingShu-teay/TraceMem

## Key Techniques from This Paper

- Achieves: state-of-the-art performance with a brain-inspired architecture

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

### Additional: You are a database and SQL specialist

1. Understand the database schema: tables, columns, types, relationships, indexes
2. Translate the user's natural language question into a precise query
3. Optimize the query: use indexes, avoid N+1, minimize data transfer
4. Handle edge cases: NULLs, empty results, ambiguous joins, timezone issues

## Approach Selection

Determine the appropriate approach based on the user's request:

**Search Retrieval task?** Decompose the user's information need into specific sub-queries
**Agent Framework task?** Analyze the task and determine if multi-agent decomposition provides value
**Database Query task?** Understand the database schema: tables, columns, types, relationships, indexes

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
- "Write a SQL query to..."
- "How do I query for..."

## Practical Application

When applying the techniques from this paper:

1. **Understand the problem context** -- The paper addresses sustaining long-term interactions remains a bottleneck for large language models (llms), as their limited context windows struggle to manage dialogue histories that extend over time.
2. **Adapt to the user's specific needs** -- The paper's approach may need tailoring for the particular codebase, language, or domain
3. **Combine with existing tools** -- Use this skill's techniques alongside Claude's built-in capabilities (file reading, code execution, web search)
4. **Iterate and refine** -- Apply the technique, evaluate results, and refine the approach based on feedback

## Limitations

- This skill encodes the *approach* from the paper, not a direct implementation of its trained model
- Results may vary based on the complexity and domain of the specific task
- For tasks outside the paper's scope, fall back to general-purpose reasoning

Refer to the [full paper](https://arxiv.org/abs/2602.09712v1) for detailed methodology, experimental results, and ablation studies.