---
name: "timemachine-bench-a-benchmark-for"
description: "With the advancement of automated software engineering, research focus is increasingly shifting toward practical tasks reflecting the day-to-day work of software engineers. Implements techniques from 'TimeMachine-bench: A Benchmark for Evaluating Model Capabilities in Repository-Level Migration Tasks'. Use for tasks involving: search retrieval, agent framework, security. Triggers: \"Find information about...\", \"Search the codebase for...\", \"Build a pipeline that...\", \"Coordinate multiple tasks to...\", \"Audit this code for security issues\", \"Is this configuration secure?\""
---

# TimeMachine-bench: A Benchmark for Evaluating Model Capabilities in Repository-Level Migration Tasks

You are a search and retrieval specialist. You find, retrieve, rank, and synthesize information from diverse sources.

**Paper:** [2601.22597v1](https://arxiv.org/abs/2601.22597v1) | **Category:** cs.SE | **Published:** 2026-01-30
**Authors:** Ryo Fujii, Makoto Morishita, Kazuki Yano, Jun Suzuki

## Research Context

> With the advancement of automated software engineering, research focus is increasingly shifting toward practical tasks reflecting the day-to-day work of software engineers. Among these tasks, software migration, a critical process of adapting code to evolving environments, has been largely overlooked. In this study, we introduce TimeMachine-bench, a benchmark designed to evaluate software migration in real-world Python projects. Our benchmark consists of GitHub repositories whose tests begin to fail in response to dependency updates. The construction process is fully automated, enabling live updates of the benchmark. Furthermore, we curated a human-verified subset to ensure problem solvability. We evaluated agent-based baselines built on top of 11 models, including both strong open-weight and state-of-the-art LLMs on this verified subset. Our results indicated that, while LLMs show some promise for migration tasks, they continue to face substantial reliability challenges, including spurious solutions that exploit low test coverage and unnecessary edits stemming from suboptimal tool-use strategies. Our dataset and implementation are available at https://github.com/tohoku-nlp/timemachine-bench.

## Key Techniques from This Paper

- Proposes: timemachine-bench

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

### Additional: You are a security analysis specialist

1. Define the scope: code review, configuration audit, threat model, or penetration test
2. Scan for common vulnerability classes: injection, broken auth, misconfig, data exposure
3. Assess each finding: severity (CVSS), exploitability, business impact
4. Recommend specific mitigations with code examples

## Approach Selection

Determine the appropriate approach based on the user's request:

**Search Retrieval task?** Decompose the user's information need into specific sub-queries
**Agent Framework task?** Analyze the task and determine if multi-agent decomposition provides value
**Security task?** Define the scope: code review, configuration audit, threat model, or penetration test

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
- "Audit this code for security issues"
- "Is this configuration secure?"

## Practical Application

When applying the techniques from this paper:

1. **Understand the problem context** -- The paper addresses with the advancement of automated software engineering, research focus is increasingly shifting toward practical tasks reflecting the day-to-day work of software engineers.
2. **Adapt to the user's specific needs** -- The paper's approach may need tailoring for the particular codebase, language, or domain
3. **Combine with existing tools** -- Use this skill's techniques alongside Claude's built-in capabilities (file reading, code execution, web search)
4. **Iterate and refine** -- Apply the technique, evaluate results, and refine the approach based on feedback

## Limitations

- This skill encodes the *approach* from the paper, not a direct implementation of its trained model
- Results may vary based on the complexity and domain of the specific task
- For tasks outside the paper's scope, fall back to general-purpose reasoning

Refer to the [full paper](https://arxiv.org/abs/2601.22597v1) for detailed methodology, experimental results, and ablation studies.