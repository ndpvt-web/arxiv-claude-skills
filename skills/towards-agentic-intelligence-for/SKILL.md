---
name: "towards-agentic-intelligence-for"
description: "The convergence of artificial intelligence and materials science presents a transformative opportunity, but achieving true acceleration in discovery requires moving beyond task-isolated, fine-tuned models toward agentic systems that plan, act, and... Implements techniques from 'Towards Agentic Intelligence for Materials Science'. Use for tasks involving: agent framework, prompt engineering. Triggers: \"Build a pipeline that...\", \"Coordinate multiple tasks to...\", \"Optimize this prompt\", \"Design a prompt for...\""
---

# Towards Agentic Intelligence for Materials Science

You are a multi-agent orchestration specialist. You decompose complex tasks, coordinate parallel agents, and aggregate results reliably.

**Paper:** [2602.00169v2](https://arxiv.org/abs/2602.00169v2) | **Category:** cond-mat.mtrl-sci | **Published:** 2026-01-29
**Authors:** Huan Zhang, Yizhan Li, Wenhao Huang, Ziyu Hou, Yu Song

## Research Context

> The convergence of artificial intelligence and materials science presents a transformative opportunity, but achieving true acceleration in discovery requires moving beyond task-isolated, fine-tuned models toward agentic systems that plan, act, and learn across the full discovery loop. This survey advances a unique pipeline-centric view that spans from corpus curation and pretraining, through domain adaptation and instruction tuning, to goal-conditioned agents interfacing with simulation and experimental platforms. Unlike prior reviews, we treat the entire process as an end-to-end system to be optimized for tangible discovery outcomes rather than proxy benchmarks. This perspective allows us to trace how upstream design choices-such as data curation and training objectives-can be aligned with downstream experimental success through effective credit assignment.   To bridge communities and establish a shared frame of reference, we first present an integrated lens that aligns terminology, evaluation, and workflow stages across AI and materials science. We then analyze the field through two focused lenses: From the AI perspective, the survey details LLM strengths in pattern recognition, predictive analytics, and natural language processing for literature mining, materials characterization, and property prediction; from the materials science perspective, it highlights applications in materials design, process optimization, and the acceleration of computational workflows via integration with external tools (e.g., DFT, robotic labs). Finally, we contrast passive, reactive approaches with agentic design, cataloging current contributions while motivating systems that pursue long-horizon goals with autonomy, memory, and tool use. This survey charts a practical roadmap towards autonomous, safety-aware LLM agents aimed at discovering novel and useful materials.

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

## Approach Selection

Determine the appropriate approach based on the user's request:

**Agent Framework task?** Analyze the task and determine if multi-agent decomposition provides value
**Prompt Engineering task?** Understand the target task and success criteria

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

## Practical Application

When applying the techniques from this paper:

1. **Understand the problem context** -- The paper addresses the convergence of artificial intelligence and materials science presents a transformative opportunity, but achieving true acceleration in discovery requires moving beyond task-isolated, fine-tuned models toward agentic systems that plan, act, and...
2. **Adapt to the user's specific needs** -- The paper's approach may need tailoring for the particular codebase, language, or domain
3. **Combine with existing tools** -- Use this skill's techniques alongside Claude's built-in capabilities (file reading, code execution, web search)
4. **Iterate and refine** -- Apply the technique, evaluate results, and refine the approach based on feedback

## Limitations

- This skill encodes the *approach* from the paper, not a direct implementation of its trained model
- Results may vary based on the complexity and domain of the specific task
- For tasks outside the paper's scope, fall back to general-purpose reasoning

Refer to the [full paper](https://arxiv.org/abs/2602.00169v2) for detailed methodology, experimental results, and ablation studies.