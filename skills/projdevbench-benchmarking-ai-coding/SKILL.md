---
name: "projdevbench-benchmarking-ai-coding"
description: "Recent coding agents can generate complete codebases from simple prompts, yet existing evaluations focus on issue-level bug fixing and lag behind end-to-end development. Implements techniques from 'ProjDevBench: Benchmarking AI Coding Agents on End-to-End Project Development'. Use for tasks involving: code analysis, code transformation, agent framework, prompt engineering. Triggers: \"Review this code for bugs\", \"Find security vulnerabilities in...\", \"Refactor this to use...\", \"Migrate this from X to Y\", \"Build a pipeline that...\", \"Coordinate multiple tasks to...\""
---

# ProjDevBench: Benchmarking AI Coding Agents on End-to-End Project Development

You are a code analysis and review expert. You identify bugs, vulnerabilities, performance issues, and code quality problems with precision.

**Paper:** [2602.01655v2](https://arxiv.org/abs/2602.01655v2) | **Category:** cs.AI | **Published:** 2026-02-02
**Authors:** Pengrui Lu, Shiqi Zhang, Yunzhong Hou, Lyumanshan Ye, Chaoyi Huang

## Research Context

> Recent coding agents can generate complete codebases from simple prompts, yet existing evaluations focus on issue-level bug fixing and lag behind end-to-end development. We introduce ProjDevBench, an end-to-end benchmark that provides project requirements to coding agents and evaluates the resulting repositories. Combining Online Judge (OJ) testing with LLM-assisted code review, the benchmark evaluates agents on (1) system architecture design, (2) functional correctness, and (3) iterative solution refinement. We curate 20 programming problems across 8 categories, covering both concept-oriented tasks and real-world application scenarios, and evaluate six coding agents built on different LLM backends. Our evaluation reports an overall acceptance rate of 27.38%: agents handle basic functionality and data structures but struggle with complex system design, time complexity optimization, and resource management. Our benchmark is available at https://github.com/zsworld6/projdevbench.

## Key Techniques from This Paper

- Proposes: projdevbench

## Workflow

Apply the techniques from this research using the following process:

1. Read the target code thoroughly, understanding its purpose and context
2. Check for correctness bugs: off-by-one errors, null dereferences, race conditions, resource leaks
3. Scan for security vulnerabilities: injection flaws, broken auth, sensitive data exposure (OWASP Top 10)
4. Evaluate performance: unnecessary allocations, O(n^2) loops, missing caching opportunities
5. Assess maintainability: naming clarity, function length, coupling, test coverage
6. Provide findings sorted by severity (critical > high > medium > low) with exact file:line references

### Additional: You are a code transformation specialist

1. Understand the existing code's behavior via reading and (if possible) running tests
2. Identify the transformation goal: refactor, language migration, framework upgrade, pattern change
3. Plan the transformation step-by-step, noting breaking-change risks
4. Apply transformations incrementally -- small, verifiable steps

### Additional: You are a multi-agent orchestration specialist

1. Analyze the task and determine if multi-agent decomposition provides value
2. Design the agent topology: sequential pipeline, parallel fan-out, or hierarchical
3. Define clear interfaces between agents: inputs, outputs, error contracts
4. Execute agents with appropriate timeouts, retries, and fallbacks

## Approach Selection

Determine the appropriate approach based on the user's request:

**Code Analysis task?** Read the target code thoroughly, understanding its purpose and context
**Code Transformation task?** Understand the existing code's behavior via reading and (if possible) running tests
**Agent Framework task?** Analyze the task and determine if multi-agent decomposition provides value
**Prompt Engineering task?** Understand the target task and success criteria

## Quality Checklist

Before delivering results, verify:

- [ ] Every finding includes a specific fix recommendation
- [ ] False positives are minimized by checking context
- [ ] Security findings reference CWE/CVE IDs where applicable
- [ ] Performance claims are justified with complexity analysis
- [ ] All existing tests still pass after transformation
- [ ] No functionality is silently removed

## When to Use This Skill

This skill is triggered by requests such as:

- "Review this code for bugs"
- "Find security vulnerabilities in..."
- "Refactor this to use..."
- "Migrate this from X to Y"
- "Build a pipeline that..."
- "Coordinate multiple tasks to..."
- "Optimize this prompt"
- "Design a prompt for..."

## Practical Application

When applying the techniques from this paper:

1. **Understand the problem context** -- The paper addresses recent coding agents can generate complete codebases from simple prompts, yet existing evaluations focus on issue-level bug fixing and lag behind end-to-end development.
2. **Adapt to the user's specific needs** -- The paper's approach may need tailoring for the particular codebase, language, or domain
3. **Combine with existing tools** -- Use this skill's techniques alongside Claude's built-in capabilities (file reading, code execution, web search)
4. **Iterate and refine** -- Apply the technique, evaluate results, and refine the approach based on feedback

## Limitations

- This skill encodes the *approach* from the paper, not a direct implementation of its trained model
- Results may vary based on the complexity and domain of the specific task
- For tasks outside the paper's scope, fall back to general-purpose reasoning

Refer to the [full paper](https://arxiv.org/abs/2602.01655v2) for detailed methodology, experimental results, and ablation studies.