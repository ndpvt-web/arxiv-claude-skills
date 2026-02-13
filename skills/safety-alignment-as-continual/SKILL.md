---
name: "safety-alignment-as-continual"
description: "Large Language Models (LLMs) often incur an alignment tax: safety post-training can reduce general utility (e.g., reasoning and coding) Implements the Safety Alignment as Continual Learning approach. Use for: agent-framework. Triggers: 'orchestrate...', 'build a pipeline...'"
---

# Safety Alignment as Continual Learning: Mitigating the Alignment Tax via Orthogonal Gradient Projection

This skill implements the approach described in *Safety Alignment as Continual Learning: Mitigating the Alignment Tax via Orthogonal Gradient Projection*. We propose Orthogonal Gradient Projection for Safety Alignment (OGPSA), a lightweight method that mitigates interference by constraining each safety update to be orthogonal (in a first-order sense) to a learned subspace capturing general capabilities.

**Paper:** [https://arxiv.org/abs/2602.07892v1](https://arxiv.org/abs/2602.07892v1) | **Category:** cs.LG | **Published:** 2026-02-08
**Code:** [https://github.com/SunGL001/OGPSA}{OGPSA}](https://github.com/SunGL001/OGPSA}{OGPSA})

## When to Use

- When orchestrating multiple steps or agents to solve a complex problem
- When facing the challenge described in the paper: accordingly, we cast safety alignment as a continual learning (cl) problem that must balance plasticity (acquiring safety constraints) and stability (preserving general abilities).

## Core Technique

**The Problem:** Accordingly, we cast safety alignment as a continual learning (CL) problem that must balance plasticity (acquiring safety constraints) and stability (preserving general abilities).

We propose Orthogonal Gradient Projection for Safety Alignment (OGPSA), a lightweight method that mitigates interference by constraining each safety update to be orthogonal (in a first-order sense) to a learned subspace capturing general capabilities.

**Key Results:** Across Supervised Fine-Tuning (SFT), Direct Preference Optimization (DPO), and sequential SFT$\rightarrow$DPO settings, OGPSA consistently improves the safety--utility Pareto frontier over standard baselines.

## Step-by-Step Workflow

1. Analyze the task to determine if decomposition into subtasks provides value
2. Apply the Safety Alignment as Continual Learning decomposition strategy to break the task into independent units
3. Design the agent topology: determine which subtasks can run in parallel vs sequentially
4. Define clear input/output contracts for each subtask (what goes in, what comes out)
5. Execute subtasks with appropriate error handling -- if one fails, others should still complete
6. Apply the paper's aggregation method to combine partial results into a unified answer
7. Validate the combined result for consistency and completeness
8. Present the final output with provenance tracking (which step produced what)

## Examples

**Example 1: Applying Safety Alignment as Continual Learning**

```
User: Help me apply the Safety Alignment as Continual Learning approach to my problem

Approach:
1. Understand the specific problem context and constraints
2. Map the problem to Safety Alignment as Continual Learning's framework
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
- Read the full problem description before applying Safety Alignment as Continual Learning
- Start with the simplest application of the technique and add complexity incrementally
- Validate intermediate results at each step of the workflow
- Adapt the approach to the specific domain rather than applying it rigidly

**Avoid:**
- Applying the technique blindly without understanding the problem context
- Skipping validation steps to save time -- errors compound quickly
- Over-engineering the solution beyond what the task requires
- Ignoring the paper's stated limitations and applying it outside its scope

## Error Handling

- **Technique doesn't apply**: If the problem doesn't match Safety Alignment as Continual Learning's assumptions, fall back to general-purpose reasoning and explain why
- **Partial results**: If some steps succeed but others fail, present the partial results with clear indication of what's missing
- **Conflicting information**: When sources disagree, present both sides with evidence and let the user decide
- **Performance issues**: If the approach is too slow, simplify by reducing the number of decomposition steps

## Limitations

- This skill encodes the *methodology* from the paper, not a trained model -- results depend on Claude's general capabilities
- The paper was evaluated on specific benchmarks; real-world tasks may differ significantly
- The original problem context: accordingly, we cast safety alignment as a continual learning (cl) problem that must balance plasticity (acquiring safety constraints) and stability (preserving general abilities)
- For tasks clearly outside the paper's scope, prefer general-purpose approaches

## Reference

**[Safety Alignment as Continual Learning: Mitigating the Alignment Tax via Orthogonal Gradient Projection](https://arxiv.org/abs/2602.07892v1)**
Key finding: Across Supervised Fine-Tuning (SFT), Direct Preference Optimization (DPO), and sequential SFT$\rightarrow$DPO settings, OGPSA consistently improves the safety--utility Pareto frontier over standard baselines.
Implementation: [https://github.com/SunGL001/OGPSA}{OGPSA}](https://github.com/SunGL001/OGPSA}{OGPSA})
Look for: methodology section, experimental setup, and ablation studies for tuning guidance.