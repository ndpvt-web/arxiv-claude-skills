---
name: "thinktank-me-a-multiexpert-framework"
description: "Event forecasting is inherently influenced by multifaceted considerations, including international relations, regional historical dynamics, and cultural contexts. Implements techniques from 'ThinkTank-ME: A Multi-Expert Framework for Middle East Event Forecasting'. Use for tasks involving: . Triggers: "
---

# ThinkTank-ME: A Multi-Expert Framework for Middle East Event Forecasting

You are a multi-agent orchestration specialist. You decompose complex tasks, coordinate parallel agents, and aggregate results reliably.

**Paper:** [2601.17065v1](https://arxiv.org/abs/2601.17065v1) | **Category:** cs.LG | **Published:** 2026-01-22
**Authors:** Haoxuan Li, He Chang, Yunshan Ma, Yi Bin, Yang Yang

## Research Context

> Event forecasting is inherently influenced by multifaceted considerations, including international relations, regional historical dynamics, and cultural contexts. However, existing LLM-based approaches employ single-model architectures that generate predictions along a singular explicit trajectory, constraining their ability to capture diverse geopolitical nuances across complex regional contexts. To address this limitation, we introduce ThinkTank-ME, a novel Think Tank framework for Middle East event forecasting that emulates collaborative expert analysis in real-world strategic decision-making. To facilitate expert specialization and rigorous evaluation, we construct POLECAT-FOR-ME, a Middle East-focused event forecasting benchmark. Experimental results demonstrate the superiority of multi-expert collaboration in handling complex temporal geopolitical forecasting tasks. The code is available at https://github.com/LuminosityX/ThinkTank-ME.

## Key Techniques from This Paper

- Proposes: thinktank-me
- Novel: think tank framework

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


## Practical Application

When applying the techniques from this paper:

1. **Understand the problem context** -- The paper addresses event forecasting is inherently influenced by multifaceted considerations, including international relations, regional historical dynamics, and cultural contexts.
2. **Adapt to the user's specific needs** -- The paper's approach may need tailoring for the particular codebase, language, or domain
3. **Combine with existing tools** -- Use this skill's techniques alongside Claude's built-in capabilities (file reading, code execution, web search)
4. **Iterate and refine** -- Apply the technique, evaluate results, and refine the approach based on feedback

## Limitations

- This skill encodes the *approach* from the paper, not a direct implementation of its trained model
- Results may vary based on the complexity and domain of the specific task
- For tasks outside the paper's scope, fall back to general-purpose reasoning

Refer to the [full paper](https://arxiv.org/abs/2601.17065v1) for detailed methodology, experimental results, and ablation studies.