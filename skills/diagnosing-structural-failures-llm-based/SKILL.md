---
name: "diagnosing-structural-failures-llm-based"
description: "Systematic reviews and meta-analyses rely on converting narrative articles into structured, numerically grounded study records Implements the Diagnosing Structural Failures in LLM-Based Evidence Extraction for Meta-Analysis approach. Use for: data-processing, database-query. Triggers: 'parse this data...', 'extract from...', 'write a SQL query...', 'query the database for...'"
---

# Diagnosing Structural Failures in LLM-Based Evidence Extraction for Meta-Analysis

This skill implements the approach described in *Diagnosing Structural Failures in LLM-Based Evidence Extraction for Meta-Analysis*. We propose a structural, diagnostic framework that evaluates LLM-based evidence extraction as a progression of schema-constrained queries with increasing relational and numerical complexity, enabling precise identification of failure points beyond atom-level extraction.

**Paper:** [https://arxiv.org/abs/2602.10881v1](https://arxiv.org/abs/2602.10881v1) | **Category:** cs.CL | **Published:** 2026-02-11
**Code:** [https://github.com/zhiyintan/LLM-Meta-Analysis](https://github.com/zhiyintan/LLM-Meta-Analysis)

## When to Use

- When extracting, cleaning, or transforming data from various formats
- When writing or optimizing database queries
- When facing the challenge described in the paper: despite rapid advances in large language models (llms), it remains unclear whether they can meet the structural requirements of this process, which hinge on preserving roles, methods, and effect-size attribution across documents rather than on recognizing isolated entities.

## Core Technique

**The Problem:** Despite rapid advances in large language models (LLMs), it remains unclear whether they can meet the structural requirements of this process, which hinge on preserving roles, methods, and effect-size attribution across documents rather than on recognizing isolated entities.

We propose a structural, diagnostic framework that evaluates LLM-based evidence extraction as a progression of schema-constrained queries with increasing relational and numerical complexity, enabling precise identification of failure points beyond atom-level extraction.

**Key Results:** Using a manually curated corpus spanning five scientific domains, together with a unified query suite and evaluation protocol, we evaluate two state-of-the-art LLMs under both per-document and long-context, multi-document input regimes.

## Step-by-Step Workflow

1. Analyze the task to determine if decomposition into subtasks provides value
2. Apply the Diagnosing Structural Failures in LLM-Based Evidence Extraction for Meta-Analysis decomposition strategy to break the task into independent units
3. Design the agent topology: determine which subtasks can run in parallel vs sequentially
4. Define clear input/output contracts for each subtask (what goes in, what comes out)
5. Execute subtasks with appropriate error handling -- if one fails, others should still complete
6. Apply the paper's aggregation method to combine partial results into a unified answer
7. Validate the combined result for consistency and completeness
8. Present the final output with provenance tracking (which step produced what)

## Examples

**Example 1: Applying Diagnosing Structural Failures in LLM-Based Evidence Extraction for Meta-Analysis**

```
User: Help me apply the Diagnosing Structural Failures in LLM-Based Evidence Extraction for Meta-Analysis approach to my problem

Approach:
1. Understand the specific problem context and constraints
2. Map the problem to Diagnosing Structural Failures in LLM-Based Evidence Extraction for Meta-Analysis's framework
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
- Read the full problem description before applying Diagnosing Structural Failures in LLM-Based Evidence Extraction for Meta-Analysis
- Start with the simplest application of the technique and add complexity incrementally
- Validate intermediate results at each step of the workflow
- Adapt the approach to the specific domain rather than applying it rigidly

**Avoid:**
- Applying the technique blindly without understanding the problem context
- Skipping validation steps to save time -- errors compound quickly
- Over-engineering the solution beyond what the task requires
- Ignoring the paper's stated limitations and applying it outside its scope

## Error Handling

- **Technique doesn't apply**: If the problem doesn't match Diagnosing Structural Failures in LLM-Based Evidence Extraction for Meta-Analysis's assumptions, fall back to general-purpose reasoning and explain why
- **Partial results**: If some steps succeed but others fail, present the partial results with clear indication of what's missing
- **Conflicting information**: When sources disagree, present both sides with evidence and let the user decide
- **Performance issues**: If the approach is too slow, simplify by reducing the number of decomposition steps

## Limitations

- This skill encodes the *methodology* from the paper, not a trained model -- results depend on Claude's general capabilities
- The paper was evaluated on specific benchmarks; real-world tasks may differ significantly
- The original problem context: despite rapid advances in large language models (llms), it remains unclear whether they can meet the structural requirements of this process, which hinge on preserving roles, methods, and effect-size attribution across documents rather than on recognizing isolated entities
- For tasks clearly outside the paper's scope, prefer general-purpose approaches

## Reference

**[Diagnosing Structural Failures in LLM-Based Evidence Extraction for Meta-Analysis](https://arxiv.org/abs/2602.10881v1)**
Key finding: Using a manually curated corpus spanning five scientific domains, together with a unified query suite and evaluation protocol, we evaluate two state-of-the-art LLMs under both per-document and long-context, multi-document input regimes.
Implementation: [https://github.com/zhiyintan/LLM-Meta-Analysis](https://github.com/zhiyintan/LLM-Meta-Analysis)
Look for: methodology section, experimental setup, and ablation studies for tuning guidance.