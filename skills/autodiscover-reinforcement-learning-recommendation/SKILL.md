---
name: "autodiscover-reinforcement-learning-recommendation"
description: "Systematic literature reviews (SLRs) are fundamental to evidence-based research, but manual screening is an increasing bottleneck as scientific output grows Implements the Autodiscover approach. Use for: data-processing, search-retrieval, agent-framework, security. Triggers: 'parse this data...', 'extract from...', 'search for...', 'find information about...', 'orchestrate...', 'build a pipeline...'"
---

# Autodiscover: A reinforcement learning recommendation system for the cold-start imbalance challenge in active learning, powered by graph-aware thompson sampling

This skill implements the approach described in *Autodiscover: A reinforcement learning recommendation system for the cold-start imbalance challenge in active learning, powered by graph-aware thompson sampling*. Systematic literature reviews (SLRs) are fundamental to evidence-based research, but manual screening is an increasing bottleneck as scientific output grows

**Paper:** [https://arxiv.org/abs/2602.05087v1](https://arxiv.org/abs/2602.05087v1) | **Category:** cs.LG | **Published:** 2026-02-04

## When to Use

- When extracting, cleaning, or transforming data from various formats
- When searching, retrieving, and synthesizing information from multiple sources
- When orchestrating multiple steps or agents to solve a complex problem
- When auditing code or systems for security vulnerabilities
- When facing the challenge described in the paper: traditional active learning (al) systems help, yet typically rely on fixed query strategies for selecting the next unlabeled documents.

## Core Technique

**The Problem:** Traditional active learning (AL) systems help, yet typically rely on fixed query strategies for selecting the next unlabeled documents.

Systematic literature reviews (SLRs) are fundamental to evidence-based research, but manual screening is an increasing bottleneck as scientific output grows. Screening features low prevalence of relevant studies and scarce, costly expert decisions. Traditional active learning (AL) systems help, yet typically rely on fixed query strategies for selecting the next unlabeled documents. These static strategies do not adapt over time and ignore the relational structure of scientific literature networks. This thesis introduces AutoDiscover, a framework that reframes AL as an online decision-making problem driven by an adaptive agent. Literature is modeled as a heterogeneous graph capturing relationships among documents, authors, and metadata. A Heterogeneous Graph Attention Network (HAN) learns node representations, which a Discounted Thompson Sampling (DTS) agent uses to dynamically manage a portfolio of query strategies. With real-time human-in-the-loop labels, the agent balances exploration and exploitation under non-stationary review dynamics, where strategy utility changes over time. On the 26-dataset SYNERGY benchmark, AutoDiscover achieves higher screening efficiency than static AL baselines. Crucially, the agent mitigates cold start by bootstrapping discovery from minimal initial labels where static approaches fail. We also introduce TS-Insight, an open-source visual analytics dashboard to interpret, verify, and diagnose the agent's decisions. Together, these contributions accelerate SLR screening under scarce expert labels and low prevalence of relevant studies.

**Key Results:** On the 26-dataset SYNERGY benchmark, AutoDiscover achieves higher screening efficiency than static AL baselines.

## Step-by-Step Workflow

1. Analyze the task to determine if decomposition into subtasks provides value
2. Apply the Autodiscover decomposition strategy to break the task into independent units
3. Design the agent topology: determine which subtasks can run in parallel vs sequentially
4. Define clear input/output contracts for each subtask (what goes in, what comes out)
5. Execute subtasks with appropriate error handling -- if one fails, others should still complete
6. Apply the paper's aggregation method to combine partial results into a unified answer
7. Validate the combined result for consistency and completeness
8. Present the final output with provenance tracking (which step produced what)

## Examples

**Example 1: Applying Autodiscover**

```
User: Help me apply the Autodiscover approach to my problem

Approach:
1. Understand the specific problem context and constraints
2. Map the problem to Autodiscover's framework
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
- Read the full problem description before applying Autodiscover
- Start with the simplest application of the technique and add complexity incrementally
- Validate intermediate results at each step of the workflow
- Adapt the approach to the specific domain rather than applying it rigidly

**Avoid:**
- Applying the technique blindly without understanding the problem context
- Skipping validation steps to save time -- errors compound quickly
- Over-engineering the solution beyond what the task requires
- Ignoring the paper's stated limitations and applying it outside its scope

## Error Handling

- **Technique doesn't apply**: If the problem doesn't match Autodiscover's assumptions, fall back to general-purpose reasoning and explain why
- **Partial results**: If some steps succeed but others fail, present the partial results with clear indication of what's missing
- **Conflicting information**: When sources disagree, present both sides with evidence and let the user decide
- **Performance issues**: If the approach is too slow, simplify by reducing the number of decomposition steps

## Limitations

- This skill encodes the *methodology* from the paper, not a trained model -- results depend on Claude's general capabilities
- The paper was evaluated on specific benchmarks; real-world tasks may differ significantly
- The original problem context: traditional active learning (al) systems help, yet typically rely on fixed query strategies for selecting the next unlabeled documents
- For tasks clearly outside the paper's scope, prefer general-purpose approaches

## Reference

**[Autodiscover: A reinforcement learning recommendation system for the cold-start imbalance challenge in active learning, powered by graph-aware thompson sampling](https://arxiv.org/abs/2602.05087v1)**
Key finding: On the 26-dataset SYNERGY benchmark, AutoDiscover achieves higher screening efficiency than static AL baselines.
Look for: methodology section, experimental setup, and ablation studies for tuning guidance.