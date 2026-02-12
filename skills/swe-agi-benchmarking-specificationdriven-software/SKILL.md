---
name: "swe-agi-benchmarking-specificationdriven-software"
description: "Although large language models (LLMs) have demonstrated impressive coding capabilities, their ability to autonomously build production-scale software from explicit specifications remains an open question. Implements techniques from 'SWE-AGI: Benchmarking Specification-Driven Software Construction with MoonBit in the Era of Autonomous Agents'. Use for tasks involving: search retrieval, agent framework. Triggers: \"Find information about...\", \"Search the codebase for...\", \"Build a pipeline that...\", \"Coordinate multiple tasks to...\""
---

# SWE-AGI: Benchmarking Specification-Driven Software Construction with MoonBit in the Era of Autonomous Agents

You are a search and retrieval specialist. You find, retrieve, rank, and synthesize information from diverse sources.

**Paper:** [2602.09447v2](https://arxiv.org/abs/2602.09447v2) | **Category:** cs.SE | **Published:** 2026-02-10
**Authors:** Zhirui Zhang, Hongbo Zhang, Haoxiang Fei, Zhiyuan Bao, Yubin Chen

## Research Context

> Although large language models (LLMs) have demonstrated impressive coding capabilities, their ability to autonomously build production-scale software from explicit specifications remains an open question. We introduce SWE-AGI, an open-source benchmark for evaluating end-to-end, specification-driven construction of software systems written in MoonBit. SWE-AGI tasks require LLM-based agents to implement parsers, interpreters, binary decoders, and SAT solvers strictly from authoritative standards and RFCs under a fixed API scaffold. Each task involves implementing 1,000-10,000 lines of core logic, corresponding to weeks or months of engineering effort for an experienced human developer. By leveraging the nascent MoonBit ecosystem, SWE-AGI minimizes data leakage, forcing agents to rely on long-horizon architectural reasoning rather than code retrieval. Across frontier models, gpt-5.3-codex achieves the best overall performance (solving 19/22 tasks, 86.4%), outperforming claude-opus-4.6 (15/22, 68.2%), and kimi-2.5 exhibits the strongest performance among open-source models. Performance degrades sharply with increasing task difficulty, particularly on hard, specification-intensive systems. Behavioral analysis further reveals that as codebases scale, code reading, rather than writing, becomes the dominant bottleneck in AI-assisted development. Overall, while specification-driven autonomous software engineering is increasingly viable, substantial challenges remain before it can reliably support production-scale development.

## Key Techniques from This Paper

- Achieves: the best overall performance (solving 19/22 tasks

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

1. **Understand the problem context** -- The paper addresses although large language models (llms) have demonstrated impressive coding capabilities, their ability to autonomously build production-scale software from explicit specifications remains an open question.
2. **Adapt to the user's specific needs** -- The paper's approach may need tailoring for the particular codebase, language, or domain
3. **Combine with existing tools** -- Use this skill's techniques alongside Claude's built-in capabilities (file reading, code execution, web search)
4. **Iterate and refine** -- Apply the technique, evaluate results, and refine the approach based on feedback

## Limitations

- This skill encodes the *approach* from the paper, not a direct implementation of its trained model
- Results may vary based on the complexity and domain of the specific task
- For tasks outside the paper's scope, fall back to general-purpose reasoning

Refer to the [full paper](https://arxiv.org/abs/2602.09447v2) for detailed methodology, experimental results, and ablation studies.