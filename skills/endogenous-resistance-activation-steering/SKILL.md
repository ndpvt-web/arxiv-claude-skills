---
name: "endogenous-resistance-activation-steering"
description: "Large language models can resist task-misaligned activation steering during inference, sometimes recovering mid-generation to produce improved responses even when steering remains active Implements the Endogenous Resistance to Activation Steering in Language Models approach. Use for: prompt-engineering. Triggers: 'optimize this prompt...', 'improve the prompt for...'"
---

# Endogenous Resistance to Activation Steering in Language Models

This skill implements the approach described in *Endogenous Resistance to Activation Steering in Language Models*. Large language models can resist task-misaligned activation steering during inference, sometimes recovering mid-generation to produce improved responses even when steering remains active

**Paper:** [https://arxiv.org/abs/2602.06941v1](https://arxiv.org/abs/2602.06941v1) | **Category:** cs.LG | **Published:** 2026-02-06

## When to Use

- When designing or optimizing prompts for better AI performance
- When facing the challenge described in the paper: large language models can resist task-misaligned activation steering during inference, sometimes recovering mid-generation to produce improved responses even when steering remains active.

## Core Technique

**The Problem:** Large language models can resist task-misaligned activation steering during inference, sometimes recovering mid-generation to produce improved responses even when steering remains active.

Large language models can resist task-misaligned activation steering during inference, sometimes recovering mid-generation to produce improved responses even when steering remains active. We term this Endogenous Steering Resistance (ESR). Using sparse autoencoder (SAE) latents to steer model activations, we find that Llama-3.3-70B shows substantial ESR, while smaller models from the Llama-3 and Gemma-2 families exhibit the phenomenon less frequently. We identify 26 SAE latents that activate differentially during off-topic content and are causally linked to ESR in Llama-3.3-70B. Zero-ablating these latents reduces the multi-attempt rate by 25%, providing causal evidence for dedicated internal consistency-checking circuits. We demonstrate that ESR can be deliberately enhanced through both prompting and training: meta-prompts instructing the model to self-monitor increase the multi-attempt rate by 4x for Llama-3.3-70B, and fine-tuning on self-correction examples successfully induces ESR-like behavior in smaller models. These findings have dual implications: ESR could protect against adversarial manipulation but might also interfere with beneficial safety interventions that rely on activation steering. Understanding and controlling these resistance mechanisms is important for developing transparent and controllable AI systems. Code is available at github.com/agencyenterprise/endogenous-steering-resistance.

**Key Results:** Large language models can resist task-misaligned activation steering during inference, sometimes recovering mid-generation to produce improved responses even when steering remains active.

## Step-by-Step Workflow

1. Analyze the task to determine if decomposition into subtasks provides value
2. Apply the Endogenous Resistance to Activation Steering in Language Models decomposition strategy to break the task into independent units
3. Design the agent topology: determine which subtasks can run in parallel vs sequentially
4. Define clear input/output contracts for each subtask (what goes in, what comes out)
5. Execute subtasks with appropriate error handling -- if one fails, others should still complete
6. Apply the paper's aggregation method to combine partial results into a unified answer
7. Validate the combined result for consistency and completeness
8. Present the final output with provenance tracking (which step produced what)

## Examples

**Example 1: Applying Endogenous Resistance to Activation Steering in Language Models**

```
User: Help me apply the Endogenous Resistance to Activation Steering in Language Models approach to my problem

Approach:
1. Understand the specific problem context and constraints
2. Map the problem to Endogenous Resistance to Activation Steering in Language Models's framework
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
- Read the full problem description before applying Endogenous Resistance to Activation Steering in Language Models
- Start with the simplest application of the technique and add complexity incrementally
- Validate intermediate results at each step of the workflow
- Adapt the approach to the specific domain rather than applying it rigidly

**Avoid:**
- Applying the technique blindly without understanding the problem context
- Skipping validation steps to save time -- errors compound quickly
- Over-engineering the solution beyond what the task requires
- Ignoring the paper's stated limitations and applying it outside its scope

## Error Handling

- **Technique doesn't apply**: If the problem doesn't match Endogenous Resistance to Activation Steering in Language Models's assumptions, fall back to general-purpose reasoning and explain why
- **Partial results**: If some steps succeed but others fail, present the partial results with clear indication of what's missing
- **Conflicting information**: When sources disagree, present both sides with evidence and let the user decide
- **Performance issues**: If the approach is too slow, simplify by reducing the number of decomposition steps

## Limitations

- This skill encodes the *methodology* from the paper, not a trained model -- results depend on Claude's general capabilities
- The paper was evaluated on specific benchmarks; real-world tasks may differ significantly
- The original problem context: large language models can resist task-misaligned activation steering during inference, sometimes recovering mid-generation to produce improved responses even when steering remains active
- For tasks clearly outside the paper's scope, prefer general-purpose approaches

## Reference

**[Endogenous Resistance to Activation Steering in Language Models](https://arxiv.org/abs/2602.06941v1)**
Key finding: Large language models can resist task-misaligned activation steering during inference, sometimes recovering mid-generation to produce improved responses even when steering remains active.
Look for: methodology section, experimental setup, and ablation studies for tuning guidance.