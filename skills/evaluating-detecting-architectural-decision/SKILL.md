---
name: "evaluating-detecting-architectural-decision"
description: "Architectural Decision Records (ADRs) play a central role in maintaining software architecture quality, yet many decision violations go unnoticed because projects lack both systematic documentation... Implements the Evaluating Large Language Models for Detecting Architectural Decision Violations approach. Use for: devops-automation, data-processing, agent-framework. Triggers: 'set up CI/CD...', 'create a Dockerfile...', 'parse this data...', 'extract from...', 'orchestrate...', 'build a pipeline...'"
---

# Evaluating Large Language Models for Detecting Architectural Decision Violations

This skill implements the approach described in *Evaluating Large Language Models for Detecting Architectural Decision Violations*. Architectural Decision Records (ADRs) play a central role in maintaining software architecture quality, yet many decision violations go unnoticed because projects lack both systematic documentation...

**Paper:** [https://arxiv.org/abs/2602.07609v1](https://arxiv.org/abs/2602.07609v1) | **Category:** cs.SE | **Published:** 2026-02-07

## When to Use

- When automating deployment, CI/CD, or infrastructure tasks
- When extracting, cleaning, or transforming data from various formats
- When orchestrating multiple steps or agents to solve a complex problem
- When facing the challenge described in the paper: architectural decision records (adrs) play a central role in maintaining software architecture quality, yet many decision violations go unnoticed because projects lack both systematic documentation and automated detection mechanisms.

## Core Technique

**The Problem:** Architectural Decision Records (ADRs) play a central role in maintaining software architecture quality, yet many decision violations go unnoticed because projects lack both systematic documentation and automated detection mechanisms.

Architectural Decision Records (ADRs) play a central role in maintaining software architecture quality, yet many decision violations go unnoticed because projects lack both systematic documentation and automated detection mechanisms. Recent advances in Large Language Models (LLMs) open up new possibilities for automating architectural reasoning at scale. We investigated how effectively LLMs can identify decision violations in open-source systems by examining their agreement, accuracy, and inherent limitations. Our study analyzed 980 ADRs across 109 GitHub repositories using a multi-model pipeline in which one LLM primary screens potential decision violations, and three additional LLMs independently validate the reasoning. We assessed agreement, accuracy, precision, and recall, and complemented the quantitative findings with expert evaluation. The models achieved substantial agreement and strong accuracy for explicit, code-inferable decisions. Accuracy falls short for implicit or deployment-oriented decisions that depend on deployment configuration or organizational knowledge. Therefore, LLMs can meaningfully support validation of architectural decision compliance; however, they are not yet replacing human expertise for decisions not focused on code.

**Key Results:** The models achieved substantial agreement and strong accuracy for explicit, code-inferable decisions.

## Step-by-Step Workflow

1. Analyze the task to determine if decomposition into subtasks provides value
2. Apply the Evaluating Large Language Models for Detecting Architectural Decision Violations decomposition strategy to break the task into independent units
3. Design the agent topology: determine which subtasks can run in parallel vs sequentially
4. Define clear input/output contracts for each subtask (what goes in, what comes out)
5. Execute subtasks with appropriate error handling -- if one fails, others should still complete
6. Apply the paper's aggregation method to combine partial results into a unified answer
7. Validate the combined result for consistency and completeness
8. Present the final output with provenance tracking (which step produced what)

## Examples

**Example 1: Applying Evaluating Large Language Models for Detecting Architectural Decision Violations**

```
User: Help me apply the Evaluating Large Language Models for Detecting Architectural Decision Violations approach to my problem

Approach:
1. Understand the specific problem context and constraints
2. Map the problem to Evaluating Large Language Models for Detecting Architectural Decision Violations's framework
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
- Read the full problem description before applying Evaluating Large Language Models for Detecting Architectural Decision Violations
- Start with the simplest application of the technique and add complexity incrementally
- Validate intermediate results at each step of the workflow
- Adapt the approach to the specific domain rather than applying it rigidly

**Avoid:**
- Applying the technique blindly without understanding the problem context
- Skipping validation steps to save time -- errors compound quickly
- Over-engineering the solution beyond what the task requires
- Ignoring the paper's stated limitations and applying it outside its scope

## Error Handling

- **Technique doesn't apply**: If the problem doesn't match Evaluating Large Language Models for Detecting Architectural Decision Violations's assumptions, fall back to general-purpose reasoning and explain why
- **Partial results**: If some steps succeed but others fail, present the partial results with clear indication of what's missing
- **Conflicting information**: When sources disagree, present both sides with evidence and let the user decide
- **Performance issues**: If the approach is too slow, simplify by reducing the number of decomposition steps

## Limitations

- This skill encodes the *methodology* from the paper, not a trained model -- results depend on Claude's general capabilities
- The paper was evaluated on specific benchmarks; real-world tasks may differ significantly
- The original problem context: architectural decision records (adrs) play a central role in maintaining software architecture quality, yet many decision violations go unnoticed because projects lack both systematic documentation and automated detection mechanisms
- For tasks clearly outside the paper's scope, prefer general-purpose approaches

## Reference

**[Evaluating Large Language Models for Detecting Architectural Decision Violations](https://arxiv.org/abs/2602.07609v1)**
Key finding: The models achieved substantial agreement and strong accuracy for explicit, code-inferable decisions.
Look for: methodology section, experimental setup, and ablation studies for tuning guidance.