---
name: "orch-many-analyses-one"
description: "Recent advances in large-scale language models (LLMs) have made multi-agent architectures attractive for challenging reasoning tasks Implements the ORCH approach. Use for: devops-automation, agent-framework, security. Triggers: 'set up CI/CD...', 'create a Dockerfile...', 'orchestrate...', 'build a pipeline...', 'audit for security...', 'find vulnerabilities...'"
---

# ORCH: many analyses, one merge-a deterministic multi-agent orchestrator for discrete-choice reasoning with EMA-guided routing

This skill implements the approach described in *ORCH: many analyses, one merge-a deterministic multi-agent orchestrator for discrete-choice reasoning with EMA-guided routing*. We propose ORCH, a deterministic coordination framework for discrete-choice reasoning that orchestrates heterogeneous LLMs.

**Paper:** [https://arxiv.org/abs/2602.01797v1](https://arxiv.org/abs/2602.01797v1) | **Category:** cs.AI | **Published:** 2026-02-02

## When to Use

- When automating deployment, CI/CD, or infrastructure tasks
- When orchestrating multiple steps or agents to solve a complex problem
- When auditing code or systems for security vulnerabilities
- When facing the challenge described in the paper: however, many existing systems rely on stochastic routing or ad-hoc heuristics, making their behavior difficult to reproduce and their decision process hard to interpret.

## Core Technique

**The Problem:** However, many existing systems rely on stochastic routing or ad-hoc heuristics, making their behavior difficult to reproduce and their decision process hard to interpret.

We propose ORCH, a deterministic coordination framework for discrete-choice reasoning that orchestrates heterogeneous LLMs.

**Key Results:** Experiments on MMLU, MMLU-Pro, and GSM8K show that ORCH consistently outperforms single-model baselines and a majority-vote ensemble.

## Step-by-Step Workflow

1. Analyze the task to determine if decomposition into subtasks provides value
2. Apply the ORCH decomposition strategy to break the task into independent units
3. Design the agent topology: determine which subtasks can run in parallel vs sequentially
4. Define clear input/output contracts for each subtask (what goes in, what comes out)
5. Execute subtasks with appropriate error handling -- if one fails, others should still complete
6. Apply the paper's aggregation method to combine partial results into a unified answer
7. Validate the combined result for consistency and completeness
8. Present the final output with provenance tracking (which step produced what)

## Examples

**Example 1: Applying ORCH**

```
User: Help me apply the ORCH approach to my problem

Approach:
1. Understand the specific problem context and constraints
2. Map the problem to ORCH's framework
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
- Read the full problem description before applying ORCH
- Start with the simplest application of the technique and add complexity incrementally
- Validate intermediate results at each step of the workflow
- Adapt the approach to the specific domain rather than applying it rigidly

**Avoid:**
- Applying the technique blindly without understanding the problem context
- Skipping validation steps to save time -- errors compound quickly
- Over-engineering the solution beyond what the task requires
- Ignoring the paper's stated limitations and applying it outside its scope

## Error Handling

- **Technique doesn't apply**: If the problem doesn't match ORCH's assumptions, fall back to general-purpose reasoning and explain why
- **Partial results**: If some steps succeed but others fail, present the partial results with clear indication of what's missing
- **Conflicting information**: When sources disagree, present both sides with evidence and let the user decide
- **Performance issues**: If the approach is too slow, simplify by reducing the number of decomposition steps

## Limitations

- This skill encodes the *methodology* from the paper, not a trained model -- results depend on Claude's general capabilities
- The paper was evaluated on specific benchmarks; real-world tasks may differ significantly
- The original problem context: however, many existing systems rely on stochastic routing or ad-hoc heuristics, making their behavior difficult to reproduce and their decision process hard to interpret
- For tasks clearly outside the paper's scope, prefer general-purpose approaches

## Reference

**[ORCH: many analyses, one merge-a deterministic multi-agent orchestrator for discrete-choice reasoning with EMA-guided routing](https://arxiv.org/abs/2602.01797v1)**
Key finding: Experiments on MMLU, MMLU-Pro, and GSM8K show that ORCH consistently outperforms single-model baselines and a majority-vote ensemble.
Look for: methodology section, experimental setup, and ablation studies for tuning guidance.