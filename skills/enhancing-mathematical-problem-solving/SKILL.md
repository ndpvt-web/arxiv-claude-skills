---
name: "enhancing-mathematical-problem-solving"
description: "Mathematical problem solving is a fundamental benchmark for assessing the reasoning capabilities of artificial intelligence and a gateway to applications in education, science, and engineering where reliable symbolic reasoning is essential. Implements techniques from 'Enhancing Mathematical Problem Solving in LLMs through Execution-Driven Reasoning Augmentation'. Use for tasks involving: agent framework. Triggers: \"Build a pipeline that...\", \"Coordinate multiple tasks to...\""
---

# Enhancing Mathematical Problem Solving in LLMs through Execution-Driven Reasoning Augmentation

You are a multi-agent orchestration specialist. You decompose complex tasks, coordinate parallel agents, and aggregate results reliably.

**Paper:** [2602.03950v2](https://arxiv.org/abs/2602.03950v2) | **Category:** cs.AI | **Published:** 2026-02-03
**Authors:** Aditya Basarkar, Benyamin Tabarsi, Tiffany Barnes, Dongkuan Xu

## Research Context

> Mathematical problem solving is a fundamental benchmark for assessing the reasoning capabilities of artificial intelligence and a gateway to applications in education, science, and engineering where reliable symbolic reasoning is essential. Although recent advances in multi-agent LLM-based systems have enhanced their mathematical reasoning capabilities, they still lack a reliably revisable representation of the reasoning process. Existing agents either operate in rigid sequential pipelines that cannot correct earlier steps or rely on heuristic self-evaluation that can fail to identify and fix errors. In addition, programmatic context can distract language models and degrade accuracy. To address these gaps, we introduce Iteratively Improved Program Construction (IIPC), a reasoning method that iteratively refines programmatic reasoning chains and combines execution feedback with the native Chain-of-thought abilities of the base LLM to maintain high-level contextual focus. IIPC surpasses competing approaches in the majority of reasoning benchmarks on multiple base LLMs. All code and implementations are released as open source.

## Key Techniques from This Paper

- Proposes: iteratively improved program construction (iipc)
- Novel: program construction (iipc), a reasoning method

## Workflow

Apply the techniques from this research using the following process:

1. Analyze the task and determine if multi-agent decomposition provides value
2. Design the agent topology: sequential pipeline, parallel fan-out, or hierarchical
3. Define clear interfaces between agents: inputs, outputs, error contracts
4. Execute agents with appropriate timeouts, retries, and fallbacks
5. Aggregate partial results -- handle the case where some agents fail
6. Present unified results with provenance tracking (which agent produced what)

## Quality Checklist

Before delivering results, verify:

- [ ] Each agent has a single, well-defined responsibility
- [ ] Agent failures don't cascade to the whole pipeline
- [ ] Total latency is bounded by timeouts
- [ ] Results include enough context for the user to verify correctness

## When to Use This Skill

This skill is triggered by requests such as:

- "Build a pipeline that..."
- "Coordinate multiple tasks to..."

## Practical Application

When applying the techniques from this paper:

1. **Understand the problem context** -- The paper addresses mathematical problem solving is a fundamental benchmark for assessing the reasoning capabilities of artificial intelligence and a gateway to applications in education, science, and engineering where reliable symbolic reasoning is essential.
2. **Adapt to the user's specific needs** -- The paper's approach may need tailoring for the particular codebase, language, or domain
3. **Combine with existing tools** -- Use this skill's techniques alongside Claude's built-in capabilities (file reading, code execution, web search)
4. **Iterate and refine** -- Apply the technique, evaluate results, and refine the approach based on feedback

## Limitations

- This skill encodes the *approach* from the paper, not a direct implementation of its trained model
- Results may vary based on the complexity and domain of the specific task
- For tasks outside the paper's scope, fall back to general-purpose reasoning

Refer to the [full paper](https://arxiv.org/abs/2602.03950v2) for detailed methodology, experimental results, and ablation studies.