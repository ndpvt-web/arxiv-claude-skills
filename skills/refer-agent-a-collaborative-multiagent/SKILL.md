---
name: "refer-agent-a-collaborative-multiagent"
description: "Referring Video Object Segmentation (RVOS) aims to segment objects in videos based on textual queries. Implements techniques from 'Refer-Agent: A Collaborative Multi-Agent System with Reasoning and Reflection for Referring Video Object Segmentation'. Use for tasks involving: agent framework, prompt engineering, design ui. Triggers: \"Build a pipeline that...\", \"Coordinate multiple tasks to...\", \"Optimize this prompt\", \"Design a prompt for...\", \"Build a UI for...\", \"Design a dashboard that...\""
---

# Refer-Agent: A Collaborative Multi-Agent System with Reasoning and Reflection for Referring Video Object Segmentation

You are a multi-agent orchestration specialist. You decompose complex tasks, coordinate parallel agents, and aggregate results reliably.

**Paper:** [2602.03595v2](https://arxiv.org/abs/2602.03595v2) | **Category:** cs.CV | **Published:** 2026-02-03
**Authors:** Haichao Jiang, Tianming Liang, Wei-Shi Zheng, Jian-Fang Hu

## Research Context

> Referring Video Object Segmentation (RVOS) aims to segment objects in videos based on textual queries. Current methods mainly rely on large-scale supervised fine-tuning (SFT) of Multi-modal Large Language Models (MLLMs). However, this paradigm suffers from heavy data dependence and limited scalability against the rapid evolution of MLLMs. Although recent zero-shot approaches offer a flexible alternative, their performance remains significantly behind SFT-based methods, due to the straightforward workflow designs. To address these limitations, we propose \textbf{Refer-Agent}, a collaborative multi-agent system with alternating reasoning-reflection mechanisms. This system decomposes RVOS into step-by-step reasoning process. During reasoning, we introduce a Coarse-to-Fine frame selection strategy to ensure the frame diversity and textual relevance, along with a Dynamic Focus Layout that adaptively adjusts the agent's visual focus. Furthermore, we propose a Chain-of-Reflection mechanism, which employs a Questioner-Responder pair to generate a self-reflection chain, enabling the system to verify intermediate results and generates feedback for next-round reasoning refinement. Extensive experiments on five challenging benchmarks demonstrate that Refer-Agent significantly outperforms state-of-the-art methods, including both SFT-based models and zero-shot approaches. Moreover, Refer-Agent is flexible and enables fast integration of new MLLMs without any additional fine-tuning costs. Code will be released at https://github.com/iSEE-Laboratory/Refer-Agent.

## Key Techniques from This Paper

- Proposes: \textbf{refer-agent}
- Proposes: a coarse-to-fine frame selection strategy to ensure the frame diversity and textual relevance
- Achieves: state-of-the-art methods

## Workflow

Apply the techniques from this research using the following process:

1. Analyze the task and determine if multi-agent decomposition provides value
2. Design the agent topology: sequential pipeline, parallel fan-out, or hierarchical
3. Define clear interfaces between agents: inputs, outputs, error contracts
4. Execute agents with appropriate timeouts, retries, and fallbacks
5. Aggregate partial results -- handle the case where some agents fail
6. Present unified results with provenance tracking (which agent produced what)

### Additional: You are a prompt engineering specialist

1. Understand the target task and success criteria
2. Draft an initial prompt using appropriate techniques: zero-shot, few-shot, CoT, ReAct
3. Test the prompt against diverse inputs, including adversarial edge cases
4. Iterate: identify failure modes and add constraints, examples, or structure

### Additional: You are a UI/UX design specialist

1. Understand the requirements: platform, users, brand guidelines, accessibility needs
2. Design the layout: information hierarchy, component placement, responsive breakpoints
3. Implement with modern frameworks (React, HTML/CSS, Tailwind, etc.)
4. Apply accessibility: semantic HTML, ARIA labels, keyboard navigation, contrast ratios

## Approach Selection

Determine the appropriate approach based on the user's request:

**Agent Framework task?** Analyze the task and determine if multi-agent decomposition provides value
**Prompt Engineering task?** Understand the target task and success criteria
**Design Ui task?** Understand the requirements: platform, users, brand guidelines, accessibility needs

## Quality Checklist

Before delivering results, verify:

- [ ] Each agent has a single, well-defined responsibility
- [ ] Agent failures don't cascade to the whole pipeline
- [ ] Total latency is bounded by timeouts
- [ ] Results include enough context for the user to verify correctness
- [ ] Prompt is as short as possible while maintaining performance
- [ ] Few-shot examples cover the distribution of real inputs

## When to Use This Skill

This skill is triggered by requests such as:

- "Build a pipeline that..."
- "Coordinate multiple tasks to..."
- "Optimize this prompt"
- "Design a prompt for..."
- "Build a UI for..."
- "Design a dashboard that..."

## Practical Application

When applying the techniques from this paper:

1. **Understand the problem context** -- The paper addresses referring video object segmentation (rvos) aims to segment objects in videos based on textual queries.
2. **Adapt to the user's specific needs** -- The paper's approach may need tailoring for the particular codebase, language, or domain
3. **Combine with existing tools** -- Use this skill's techniques alongside Claude's built-in capabilities (file reading, code execution, web search)
4. **Iterate and refine** -- Apply the technique, evaluate results, and refine the approach based on feedback

## Limitations

- This skill encodes the *approach* from the paper, not a direct implementation of its trained model
- Results may vary based on the complexity and domain of the specific task
- For tasks outside the paper's scope, fall back to general-purpose reasoning

Refer to the [full paper](https://arxiv.org/abs/2602.03595v2) for detailed methodology, experimental results, and ablation studies.