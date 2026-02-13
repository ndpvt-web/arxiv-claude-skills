---
name: "unaligning-single-unlabeled-prompt"
description: "Safety alignment is only as robust as its weakest failure mode Implements the GRP-Obliteration approach. Use for: devops-automation, search-retrieval, content-generation, agent-framework. Triggers: 'set up CI/CD...', 'create a Dockerfile...', 'search for...', 'find information about...', 'write documentation...', 'generate a report...'"
---

# GRP-Obliteration: Unaligning LLMs With a Single Unlabeled Prompt

This skill implements the approach described in *GRP-Obliteration: Unaligning LLMs With a Single Unlabeled Prompt*. Safety alignment is only as robust as its weakest failure mode

**Paper:** [https://arxiv.org/abs/2602.06258v1](https://arxiv.org/abs/2602.06258v1) | **Category:** cs.LG | **Published:** 2026-02-05

## When to Use

- When automating deployment, CI/CD, or infrastructure tasks
- When searching, retrieving, and synthesizing information from multiple sources
- When generating documentation, reports, or structured content
- When orchestrating multiple steps or agents to solve a complex problem
- When facing the challenge described in the paper: despite extensive work on safety post-training, it has been shown that models can be readily unaligned through post-deployment fine-tuning.

## Core Technique

**The Problem:** Despite extensive work on safety post-training, it has been shown that models can be readily unaligned through post-deployment fine-tuning.

Safety alignment is only as robust as its weakest failure mode. Despite extensive work on safety post-training, it has been shown that models can be readily unaligned through post-deployment fine-tuning. However, these methods often require extensive data curation and degrade model utility.   In this work, we extend the practical limits of unalignment by introducing GRP-Obliteration (GRP-Oblit), a method that uses Group Relative Policy Optimization (GRPO) to directly remove safety constraints from target models. We show that a single unlabeled prompt is sufficient to reliably unalign safety-aligned models while largely preserving their utility, and that GRP-Oblit achieves stronger unalignment on average than existing state-of-the-art techniques. Moreover, GRP-Oblit generalizes beyond language models and can also unalign diffusion-based image generation systems.   We evaluate GRP-Oblit on six utility benchmarks and five safety benchmarks across fifteen 7-20B parameter models, spanning instruct and reasoning models, as well as dense and MoE architectures. The evaluated model families include GPT-OSS, distilled DeepSeek, Gemma, Llama, Ministral, and Qwen.

**Key Results:** We show that a single unlabeled prompt is sufficient to reliably unalign safety-aligned models while largely preserving their utility, and that GRP-Oblit achieves stronger unalignment on average than existing state-of-the-art techniques.

## Step-by-Step Workflow

1. Analyze the task to determine if decomposition into subtasks provides value
2. Apply the GRP-Obliteration decomposition strategy to break the task into independent units
3. Design the agent topology: determine which subtasks can run in parallel vs sequentially
4. Define clear input/output contracts for each subtask (what goes in, what comes out)
5. Execute subtasks with appropriate error handling -- if one fails, others should still complete
6. Apply the paper's aggregation method to combine partial results into a unified answer
7. Validate the combined result for consistency and completeness
8. Present the final output with provenance tracking (which step produced what)

## Examples

**Example 1: Applying GRP-Obliteration**

```
User: Help me apply the GRP-Obliteration approach to my problem

Approach:
1. Understand the specific problem context and constraints
2. Map the problem to GRP-Obliteration's framework
3. Apply the technique step by step, adapting to the specific domain
4. Validate results and iterate on the approach

Output: A tailored solution applying the paper's methodology
to the user's specific context, with explanation of each step.
```

**Example 2: Debugging and iteration**

```
User: The initial approach isn't working well, can you refine it?

Approach:
1. Identify where the current approach is falling short
2. Consult the paper's ablation studies for guidance on what matters most
3. Adjust parameters or approach based on the paper's recommendations
4. Re-run and compare results

Output: An improved solution with explanation of what changed and why,
referencing the paper's findings about what factors affect performance.
```

## Best Practices

**Do:**
- Read the full problem description before applying GRP-Obliteration
- Start with the simplest application of the technique and add complexity incrementally
- Validate intermediate results at each step of the workflow
- Adapt the approach to the specific domain rather than applying it rigidly

**Avoid:**
- Applying the technique blindly without understanding the problem context
- Skipping validation steps to save time -- errors compound quickly
- Over-engineering the solution beyond what the task requires
- Ignoring the paper's stated limitations and applying it outside its scope

## Error Handling

- **Technique doesn't apply**: If the problem doesn't match GRP-Obliteration's assumptions, fall back to general-purpose reasoning and explain why
- **Partial results**: If some steps succeed but others fail, present the partial results with clear indication of what's missing
- **Conflicting information**: When sources disagree, present both sides with evidence and let the user decide
- **Performance issues**: If the approach is too slow, simplify by reducing the number of decomposition steps

## Limitations

- This skill encodes the *methodology* from the paper, not a trained model -- results depend on Claude's general capabilities
- The paper was evaluated on specific benchmarks; real-world tasks may differ significantly
- The original problem context: despite extensive work on safety post-training, it has been shown that models can be readily unaligned through post-deployment fine-tuning
- For tasks clearly outside the paper's scope, prefer general-purpose approaches

## Reference

**[GRP-Obliteration: Unaligning LLMs With a Single Unlabeled Prompt](https://arxiv.org/abs/2602.06258v1)**
Key finding: We show that a single unlabeled prompt is sufficient to reliably unalign safety-aligned models while largely preserving their utility, and that GRP-Oblit achieves stronger unalignment on average than existing state-of-the-art techniques.
Look for: methodology section, experimental setup, and ablation studies for tuning guidance.