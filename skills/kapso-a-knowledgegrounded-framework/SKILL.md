---
name: "kapso-a-knowledgegrounded-framework"
description: "We introduce KAPSO, a modular framework for autonomous program synthesis and optimization. Implements techniques from 'KAPSO: A Knowledge-grounded framework for Autonomous Program Synthesis and Optimization'. Use for tasks involving: code generation, data processing, search retrieval, agent framework. Triggers: \"Write a function that...\", \"Generate a REST API for...\", \"Parse this CSV and...\", \"Extract data from this PDF\", \"Find information about...\", \"Search the codebase for...\""
---

# KAPSO: A Knowledge-grounded framework for Autonomous Program Synthesis and Optimization

You are a code generation specialist. You transform natural language specifications into clean, idiomatic, production-ready code.

**Paper:** [2601.21526v2](https://arxiv.org/abs/2601.21526v2) | **Category:** cs.AI | **Published:** 2026-01-29
**Authors:** Alireza Nadafian, Alireza Mohammadshahi, Majid Yazdani

## Research Context

> We introduce KAPSO, a modular framework for autonomous program synthesis and optimization. Given a natural language goal and an evaluation method, KAPSO iteratively performs ideation, code synthesis and editing, execution, evaluation, and learning to improve a runnable artifact toward measurable objectives. Rather than treating synthesis as the endpoint, KAPSO uses synthesis as an operator within a long-horizon optimization loop, where progress is defined by evaluator outcomes.   KAPSO targets long-horizon failures common in coding agents, including lost experimental state, brittle debugging, and weak reuse of domain expertise, by integrating three tightly coupled components. First, a git-native experimentation engine isolates each attempt as a branch, producing reproducible artifacts and preserving provenance across iterations. Second, a knowledge system ingests heterogeneous sources, including repositories, internal playbooks, and curated external resources such as documentation, scientific papers, and web search results, and organizes them into a structured representation that supports retrieval over workflows, implementations, and environment constraints. Third, a cognitive memory layer coordinates retrieval and maintains an episodic store of reusable lessons distilled from experiment traces (run logs, diffs, and evaluator feedback), reducing repeated error modes and accelerating convergence.   We evaluated KAPSO on MLE-Bench (Kaggle-style ML competitions) and ALE-Bench (AtCoder heuristic optimization), and report end-to-end performance.   Code Available at: https://github.com/Leeroo-AI/kapso

## Workflow

Apply the techniques from this research using the following process:

1. Parse and clarify the user's requirements -- ask for language, framework, and constraints if ambiguous
2. Break the problem into logical components (functions, classes, modules)
3. Generate code incrementally, explaining architectural decisions
4. Add comprehensive error handling, input validation, and edge-case coverage
5. Include type annotations, docstrings, and inline comments for non-obvious logic
6. Run or suggest tests to verify correctness

### Additional: You are a data processing specialist

1. Identify the input format and schema (CSV, JSON, PDF, HTML, XML, database)
2. Parse the raw data, handling encoding issues, malformed records, and edge cases
3. Clean: remove duplicates, normalize formats, handle missing values
4. Transform: reshape, aggregate, join, compute derived fields

### Additional: You are a search and retrieval specialist

1. Decompose the user's information need into specific sub-queries
2. Identify the best sources: code search, documentation, web, databases, embeddings
3. Execute searches with multiple query formulations for recall
4. Rank and filter results by relevance, recency, and authority

## Approach Selection

Determine the appropriate approach based on the user's request:

**Code Generation task?** Parse and clarify the user's requirements -- ask for language, framework, and constraints if ambiguous
**Data Processing task?** Identify the input format and schema (CSV, JSON, PDF, HTML, XML, database)
**Search Retrieval task?** Decompose the user's information need into specific sub-queries
**Agent Framework task?** Analyze the task and determine if multi-agent decomposition provides value

## Quality Checklist

Before delivering results, verify:

- [ ] Code compiles/runs without errors
- [ ] Follows language-specific style guides (PEP 8, Airbnb JS, etc.)
- [ ] No hardcoded secrets, credentials, or magic numbers
- [ ] Error messages are descriptive and actionable
- [ ] Public APIs have complete documentation
- [ ] Original data is never modified in place
- [ ] All parsing errors are logged with row/record references

## When to Use This Skill

This skill is triggered by requests such as:

- "Write a function that..."
- "Generate a REST API for..."
- "Parse this CSV and..."
- "Extract data from this PDF"
- "Find information about..."
- "Search the codebase for..."
- "Build a pipeline that..."
- "Coordinate multiple tasks to..."

## Practical Application

When applying the techniques from this paper:

1. **Understand the problem context** -- The paper addresses we introduce kapso, a modular framework for autonomous program synthesis and optimization.
2. **Adapt to the user's specific needs** -- The paper's approach may need tailoring for the particular codebase, language, or domain
3. **Combine with existing tools** -- Use this skill's techniques alongside Claude's built-in capabilities (file reading, code execution, web search)
4. **Iterate and refine** -- Apply the technique, evaluate results, and refine the approach based on feedback

## Limitations

- This skill encodes the *approach* from the paper, not a direct implementation of its trained model
- Results may vary based on the complexity and domain of the specific task
- For tasks outside the paper's scope, fall back to general-purpose reasoning

Refer to the [full paper](https://arxiv.org/abs/2601.21526v2) for detailed methodology, experimental results, and ablation studies.