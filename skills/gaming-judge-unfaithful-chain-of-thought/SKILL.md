---
name: "gaming-judge-unfaithful-chain-of-thought"
description: "Large language models (LLMs) are increasingly used as judges to evaluate agent performance, particularly in non-verifiable settings where judgments rely on agent trajectories including chain-of-tho... Implements the Gaming the Judge approach. Use for: code-analysis, agent-framework, prompt-engineering, security. Triggers: 'review this code...', 'find bugs in...', 'orchestrate...', 'build a pipeline...', 'optimize this prompt...', 'improve the prompt for...'"
---

# Gaming the Judge: Unfaithful Chain-of-Thought Can Undermine Agent Evaluation

This skill implements the approach described in *Gaming the Judge: Unfaithful Chain-of-Thought Can Undermine Agent Evaluation*. Large language models (LLMs) are increasingly used as judges to evaluate agent performance, particularly in non-verifiable settings where judgments rely on agent trajectories including chain-of-tho...

**Paper:** [https://arxiv.org/abs/2601.14691v2](https://arxiv.org/abs/2601.14691v2) | **Category:** cs.AI | **Published:** 2026-01-21

## When to Use

- When analyzing code for quality issues, potential bugs, or optimization opportunities
- When orchestrating multiple steps or agents to solve a complex problem
- When designing or optimizing prompts for better AI performance
- When auditing code or systems for security vulnerabilities

## Core Technique

Large language models (LLMs) are increasingly used as judges to evaluate agent performance, particularly in non-verifiable settings where judgments rely on agent trajectories including chain-of-thought (CoT) reasoning. This paradigm implicitly assumes that the agent's CoT faithfully reflects both its internal reasoning and the underlying environment state. We show this assumption is brittle: LLM judges are highly susceptible to manipulation of agent reasoning traces. By systematically rewriting agent CoTs while holding actions and observations fixed, we demonstrate that manipulated reasoning alone can inflate false positive rates of state-of-the-art VLM judges by up to 90% across 800 trajectories spanning diverse web tasks. We study manipulation strategies spanning style-based approaches that alter only the presentation of reasoning and content-based approaches that fabricate signals of task progress, and find that content-based manipulations are consistently more effective. We evaluate prompting-based techniques and scaling judge-time compute, which reduce but do not fully eliminate susceptibility to manipulation. Our findings reveal a fundamental vulnerability in LLM-based evaluation and highlight the need for judging mechanisms that verify reasoning claims against observable evidence.

**Key Results:** By systematically rewriting agent CoTs while holding actions and observations fixed, we demonstrate that manipulated reasoning alone can inflate false positive rates of state-of-the-art VLM judges by up to 90% across 800 trajectories spanning diverse web tasks.

## Step-by-Step Workflow

1. Read the target code thoroughly, understanding its purpose, inputs, and outputs
2. Apply the Gaming the Judge analysis method systematically across the codebase
3. Check for correctness bugs: off-by-one errors, null dereferences, race conditions, resource leaks
4. Scan for security vulnerabilities using the OWASP Top 10 as a checklist
5. Evaluate performance: unnecessary allocations, quadratic loops, missing caching opportunities
6. Assess maintainability: naming clarity, function length, coupling, cohesion
7. Sort findings by severity (critical > high > medium > low) with exact file:line references
8. For each finding, provide a specific fix with code example

## Examples

**Example 1: Applying the technique to code generation**

```
User: Use the Gaming the Judge approach to generate a data processing pipeline

Approach:
1. Identify the pipeline stages from the user's description
2. Apply Gaming the Judge's decomposition to design each stage independently
3. Generate code for each stage with clear interfaces between them
4. Wire the stages together with error handling at each boundary
5. Add logging and monitoring hooks for observability

Output: A complete, runnable pipeline with clear stage separation,
error handling, and documentation for each component.
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
- Read the full problem description before applying Gaming the Judge
- Start with the simplest application of the technique and add complexity incrementally
- Validate intermediate results at each step of the workflow
- Adapt the approach to the specific domain rather than applying it rigidly

**Avoid:**
- Applying the technique blindly without understanding the problem context
- Skipping validation steps to save time -- errors compound quickly
- Over-engineering the solution beyond what the task requires
- Ignoring the paper's stated limitations and applying it outside its scope

## Error Handling

- **Technique doesn't apply**: If the problem doesn't match Gaming the Judge's assumptions, fall back to general-purpose reasoning and explain why
- **Partial results**: If some steps succeed but others fail, present the partial results with clear indication of what's missing
- **Conflicting information**: When sources disagree, present both sides with evidence and let the user decide
- **Performance issues**: If the approach is too slow, simplify by reducing the number of decomposition steps

## Limitations

- This skill encodes the *methodology* from the paper, not a trained model -- results depend on Claude's general capabilities
- The paper was evaluated on specific benchmarks; real-world tasks may differ significantly
- For tasks clearly outside the paper's scope, prefer general-purpose approaches

## Reference

**[Gaming the Judge: Unfaithful Chain-of-Thought Can Undermine Agent Evaluation](https://arxiv.org/abs/2601.14691v2)**
Key finding: By systematically rewriting agent CoTs while holding actions and observations fixed, we demonstrate that manipulated reasoning alone can inflate false positive rates of state-of-the-art VLM judges by up to 90% across 800 trajectories spanning diverse web tasks.
Look for: methodology section, experimental setup, and ablation studies for tuning guidance.