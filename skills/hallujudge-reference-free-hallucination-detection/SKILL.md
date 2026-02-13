---
name: "hallujudge-reference-free-hallucination-detection"
description: "Large Language models (LLMs) have shown strong capabilities in code review automation, such as review comment generation, yet they suffer from hallucinations -- where the generated review comments ... Implements the HalluJudge approach. Use for: code-analysis, documentation, search-retrieval, agent-framework. Triggers: 'review this code...', 'find bugs in...', 'help with...', 'search for...', 'find information about...'"
---

# HalluJudge: A Reference-Free Hallucination Detection for Context Misalignment in Code Review Automation

This skill implements the approach described in *HalluJudge: A Reference-Free Hallucination Detection for Context Misalignment in Code Review Automation*. Large Language models (LLMs) have shown strong capabilities in code review automation, such as review comment generation, yet they suffer from hallucinations -- where the generated review comments ...

**Paper:** [https://arxiv.org/abs/2601.19072v1](https://arxiv.org/abs/2601.19072v1) | **Category:** cs.SE | **Published:** 2026-01-27

## When to Use

- When analyzing code for quality issues, potential bugs, or optimization opportunities
- When searching, retrieving, and synthesizing information from multiple sources
- When orchestrating multiple steps or agents to solve a complex problem
- When facing the challenge described in the paper: large language models (llms) have shown strong capabilities in code review automation, such as review comment generation, yet they suffer from hallucinations -- where the generated review comments are ungrounded in the actual code -- poses a significant challenge to the adoption of llms in code review workflows.

## Core Technique

**The Problem:** Large Language models (LLMs) have shown strong capabilities in code review automation, such as review comment generation, yet they suffer from hallucinations -- where the generated review comments are ungrounded in the actual code -- poses a significant challenge to the adoption of LLMs in code review workflows.

Large Language models (LLMs) have shown strong capabilities in code review automation, such as review comment generation, yet they suffer from hallucinations -- where the generated review comments are ungrounded in the actual code -- poses a significant challenge to the adoption of LLMs in code review workflows. To address this, we explore effective and scalable methods for a hallucination detection in LLM-generated code review comments without the reference. In this work, we design HalluJudge that aims to assess the grounding of generated review comments based on the context alignment. HalluJudge includes four key strategies ranging from direct assessment to structured multi-branch reasoning (e.g., Tree-of-Thoughts). We conduct a comprehensive evaluation of these assessment strategies across Atlassian's enterprise-scale software projects to examine the effectiveness and cost-efficiency of HalluJudge. Furthermore, we analyze the alignment between HalluJudge's judgment and developer preference of the actual LLM-generated code review comments in the real-world production. Our results show that the hallucination assessment in HalluJudge is cost-effective with an F1 score of 0.85 and an average cost of $0.009. On average, 67% of the HalluJudge assessments are aligned with the developer preference of the actual LLM-generated review comments in the online production. Our results suggest that HalluJudge can serve as a practical safeguard to reduce developers' exposure to hallucinated comments, fostering trust in AI-assisted code reviews.

**Key Results:** Our results show that the hallucination assessment in HalluJudge is cost-effective with an F1 score of 0.85 and an average cost of $0.009.

## Step-by-Step Workflow

1. Read the target code thoroughly, understanding its purpose, inputs, and outputs
2. Apply the HalluJudge analysis method systematically across the codebase
3. Check for correctness bugs: off-by-one errors, null dereferences, race conditions, resource leaks
4. Scan for security vulnerabilities using the OWASP Top 10 as a checklist
5. Evaluate performance: unnecessary allocations, quadratic loops, missing caching opportunities
6. Assess maintainability: naming clarity, function length, coupling, cohesion
7. Sort findings by severity (critical > high > medium > low) with exact file:line references
8. For each finding, provide a specific fix with code example

## Examples

**Example 1: Applying the technique to code generation**

```
User: Use the HalluJudge approach to generate a data processing pipeline

Approach:
1. Identify the pipeline stages from the user's description
2. Apply HalluJudge's decomposition to design each stage independently
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
- Read the full problem description before applying HalluJudge
- Start with the simplest application of the technique and add complexity incrementally
- Validate intermediate results at each step of the workflow
- Adapt the approach to the specific domain rather than applying it rigidly

**Avoid:**
- Applying the technique blindly without understanding the problem context
- Skipping validation steps to save time -- errors compound quickly
- Over-engineering the solution beyond what the task requires
- Ignoring the paper's stated limitations and applying it outside its scope

## Error Handling

- **Technique doesn't apply**: If the problem doesn't match HalluJudge's assumptions, fall back to general-purpose reasoning and explain why
- **Partial results**: If some steps succeed but others fail, present the partial results with clear indication of what's missing
- **Conflicting information**: When sources disagree, present both sides with evidence and let the user decide
- **Performance issues**: If the approach is too slow, simplify by reducing the number of decomposition steps

## Limitations

- This skill encodes the *methodology* from the paper, not a trained model -- results depend on Claude's general capabilities
- The paper was evaluated on specific benchmarks; real-world tasks may differ significantly
- The original problem context: large language models (llms) have shown strong capabilities in code review automation, such as review comment generation, yet they suffer from hallucinations -- where the generated review comments are ungrounded in the actual code -- poses a significant challenge to the adoption of llms in code review workflows
- For tasks clearly outside the paper's scope, prefer general-purpose approaches

## Reference

**[HalluJudge: A Reference-Free Hallucination Detection for Context Misalignment in Code Review Automation](https://arxiv.org/abs/2601.19072v1)**
Key finding: Our results show that the hallucination assessment in HalluJudge is cost-effective with an F1 score of 0.85 and an average cost of $0.009.
Look for: methodology section, experimental setup, and ablation studies for tuning guidance.