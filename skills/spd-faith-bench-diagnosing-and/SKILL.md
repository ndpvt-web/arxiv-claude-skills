---
name: "spd-faith-bench-diagnosing-and"
description: "Chain-of-Thought reasoning is widely used to improve the interpretability of multimodal large language models (MLLMs), yet the faithfulness of the generated reasoning traces remains unclear. Implements techniques from 'SPD-Faith Bench: Diagnosing and Improving Faithfulness in Chain-of-Thought for Multimodal Large Language Models'. Use for tasks involving: agent framework. Triggers: \"Build a pipeline that...\", \"Coordinate multiple tasks to...\""
---

# SPD-Faith Bench: Diagnosing and Improving Faithfulness in Chain-of-Thought for Multimodal Large Language Models

You are a multi-agent orchestration specialist. You decompose complex tasks, coordinate parallel agents, and aggregate results reliably.

**Paper:** [2602.07833v1](https://arxiv.org/abs/2602.07833v1) | **Category:** cs.CV | **Published:** 2026-02-08
**Authors:** Weijiang Lv, Yaoxuan Feng, Xiaobo Xia, Jiayu Wang, Yan Jing

## Research Context

> Chain-of-Thought reasoning is widely used to improve the interpretability of multimodal large language models (MLLMs), yet the faithfulness of the generated reasoning traces remains unclear. Prior work has mainly focused on perceptual hallucinations, leaving reasoning level unfaithfulness underexplored. To isolate faithfulness from linguistic priors, we introduce SPD-Faith Bench, a diagnostic benchmark based on fine-grained image difference reasoning that enforces explicit visual comparison. Evaluations on state-of-the-art MLLMs reveal two systematic failure modes, perceptual blindness and perception-reasoning dissociation. We trace these failures to decaying visual attention and representation shifts in the residual stream. Guided by this analysis, we propose SAGE, a train-free visual evidence-calibrated framework that improves visual routing and aligns reasoning with perception. Our results highlight the importance of explicitly evaluating faithfulness beyond response correctness. Our benchmark and codes are available at https://github.com/Johanson-colab/SPD-Faith-Bench.

## Key Techniques from This Paper

- Proposes: spd-faith bench

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

1. **Understand the problem context** -- The paper addresses chain-of-thought reasoning is widely used to improve the interpretability of multimodal large language models (mllms), yet the faithfulness of the generated reasoning traces remains unclear.
2. **Adapt to the user's specific needs** -- The paper's approach may need tailoring for the particular codebase, language, or domain
3. **Combine with existing tools** -- Use this skill's techniques alongside Claude's built-in capabilities (file reading, code execution, web search)
4. **Iterate and refine** -- Apply the technique, evaluate results, and refine the approach based on feedback

## Limitations

- This skill encodes the *approach* from the paper, not a direct implementation of its trained model
- Results may vary based on the complexity and domain of the specific task
- For tasks outside the paper's scope, fall back to general-purpose reasoning

Refer to the [full paper](https://arxiv.org/abs/2602.07833v1) for detailed methodology, experimental results, and ablation studies.