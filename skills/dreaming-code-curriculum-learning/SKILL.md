---
name: "dreaming-code-curriculum-learning"
description: "Open-ended learning frames intelligence as emerging from continual interaction with an ever-expanding space of environments Implements the Dreaming in Code for Curriculum Learning in Open-Ended Worlds approach. Use for: agent-framework. Triggers: 'orchestrate...', 'build a pipeline...'"
---

# Dreaming in Code for Curriculum Learning in Open-Ended Worlds

This skill implements the approach described in *Dreaming in Code for Curriculum Learning in Open-Ended Worlds*. To address this, we propose Dreaming in Code (DiCode), a framework in which foundation models synthesize executable environment code to scaffold learning toward increasing competence.

**Paper:** [https://arxiv.org/abs/2602.08194v1](https://arxiv.org/abs/2602.08194v1) | **Category:** cs.LG | **Published:** 2026-02-09
**Code:** [https://github.com/konstantinosmitsides/dreaming-in-code.](https://github.com/konstantinosmitsides/dreaming-in-code.)

## When to Use

- When orchestrating multiple steps or agents to solve a complex problem
- When facing the challenge described in the paper: in complex open-ended worlds, the large combinatorial space of possible challenges makes it difficult for agents to discover sequences of experiences that remain consistently learnable.

## Core Technique

**The Problem:** In complex open-ended worlds, the large combinatorial space of possible challenges makes it difficult for agents to discover sequences of experiences that remain consistently learnable.

To address this, we propose Dreaming in Code (DiCode), a framework in which foundation models synthesize executable environment code to scaffold learning toward increasing competence.

**Key Results:** Empirically, DiCode enables agents to acquire long-horizon skills, achieving a $16\%$ improvement in mean return over the strongest baseline and non-zero success on late-game combat tasks where prior methods fail.

## Step-by-Step Workflow

1. Analyze the task to determine if decomposition into subtasks provides value
2. Apply the Dreaming in Code for Curriculum Learning in Open-Ended Worlds decomposition strategy to break the task into independent units
3. Design the agent topology: determine which subtasks can run in parallel vs sequentially
4. Define clear input/output contracts for each subtask (what goes in, what comes out)
5. Execute subtasks with appropriate error handling -- if one fails, others should still complete
6. Apply the paper's aggregation method to combine partial results into a unified answer
7. Validate the combined result for consistency and completeness
8. Present the final output with provenance tracking (which step produced what)

## Examples

**Example 1: Applying Dreaming in Code for Curriculum Learning in Open-Ended Worlds**

```
User: Help me apply the Dreaming in Code for Curriculum Learning in Open-Ended Worlds approach to my problem

Approach:
1. Understand the specific problem context and constraints
2. Map the problem to Dreaming in Code for Curriculum Learning in Open-Ended Worlds's framework
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
- Read the full problem description before applying Dreaming in Code for Curriculum Learning in Open-Ended Worlds
- Start with the simplest application of the technique and add complexity incrementally
- Validate intermediate results at each step of the workflow
- Adapt the approach to the specific domain rather than applying it rigidly

**Avoid:**
- Applying the technique blindly without understanding the problem context
- Skipping validation steps to save time -- errors compound quickly
- Over-engineering the solution beyond what the task requires
- Ignoring the paper's stated limitations and applying it outside its scope

## Error Handling

- **Technique doesn't apply**: If the problem doesn't match Dreaming in Code for Curriculum Learning in Open-Ended Worlds's assumptions, fall back to general-purpose reasoning and explain why
- **Partial results**: If some steps succeed but others fail, present the partial results with clear indication of what's missing
- **Conflicting information**: When sources disagree, present both sides with evidence and let the user decide
- **Performance issues**: If the approach is too slow, simplify by reducing the number of decomposition steps

## Limitations

- This skill encodes the *methodology* from the paper, not a trained model -- results depend on Claude's general capabilities
- The paper was evaluated on specific benchmarks; real-world tasks may differ significantly
- The original problem context: in complex open-ended worlds, the large combinatorial space of possible challenges makes it difficult for agents to discover sequences of experiences that remain consistently learnable
- For tasks clearly outside the paper's scope, prefer general-purpose approaches

## Reference

**[Dreaming in Code for Curriculum Learning in Open-Ended Worlds](https://arxiv.org/abs/2602.08194v1)**
Key finding: Empirically, DiCode enables agents to acquire long-horizon skills, achieving a $16\%$ improvement in mean return over the strongest baseline and non-zero success on late-game combat tasks where prior methods fail.
Implementation: [https://github.com/konstantinosmitsides/dreaming-in-code.](https://github.com/konstantinosmitsides/dreaming-in-code.)
Look for: methodology section, experimental setup, and ablation studies for tuning guidance.