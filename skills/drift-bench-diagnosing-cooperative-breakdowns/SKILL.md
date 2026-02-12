---
name: "drift-bench-diagnosing-cooperative-breakdowns"
description: "As Large Language Models transition to autonomous agents, user inputs frequently violate cooperative assumptions (e.g., implicit intent, missing parameters, false presuppositions, or ambiguous expr... Implements the Drift-Bench approach. Use for: search-retrieval, agent-framework, prompt-engineering. Triggers: 'search for...', 'find information about...', 'orchestrate...', 'build a pipeline...', 'optimize this prompt...', 'improve the prompt for...'"
---

# Drift-Bench: Diagnosing Cooperative Breakdowns in LLM Agents under Input Faults via Multi-Turn Interaction

This skill implements the approach described in *Drift-Bench: Diagnosing Cooperative Breakdowns in LLM Agents under Input Faults via Multi-Turn Interaction*. We introduce \textbf{Drift-Bench}, the first diagnostic benchmark that evaluates agentic pragmatics under input faults through multi-turn clarification across state-oriented and service-oriented execution environments.

**Paper:** [https://arxiv.org/abs/2602.02455v1](https://arxiv.org/abs/2602.02455v1) | **Category:** cs.AI | **Published:** 2026-02-02

## When to Use

- When searching, retrieving, and synthesizing information from multiple sources
- When orchestrating multiple steps or agents to solve a complex problem
- When designing or optimizing prompts for better AI performance

## Core Technique

We introduce \textbf{Drift-Bench}, the first diagnostic benchmark that evaluates agentic pragmatics under input faults through multi-turn clarification across state-oriented and service-oriented execution environments.

**Key Results:** Experiments show substantial performance drops under these faults, with clarification effectiveness varying across user personas and fault types.

## Step-by-Step Workflow

1. Analyze the user's query to identify the core information need and any constraints
2. Decompose the query into 2-4 specific sub-questions that can be independently searched
3. Apply the Drift-Bench approach: formulate multiple search strategies per sub-question
4. Execute searches across available sources (codebase, documentation, web, databases)
5. Rank results by relevance using the paper's scoring criteria: authority, recency, and semantic match
6. Cross-reference findings across sources to identify consensus and conflicts
7. Synthesize results into a structured answer with inline citations
8. Highlight confidence levels for each claim and flag any information gaps

## Examples

**Example 1: Multi-source information synthesis**

```
User: Research how to implement drift-bench in my project

Approach:
1. Decompose into sub-queries: architecture, implementation, configuration, testing
2. Search documentation, code examples, and best practices for each
3. Cross-reference findings to identify the consensus approach
4. Synthesize into a step-by-step implementation guide

Output: A structured research report with implementation guide,
code examples, and links to authoritative sources.
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
- Read the full problem description before applying Drift-Bench
- Start with the simplest application of the technique and add complexity incrementally
- Validate intermediate results at each step of the workflow
- Adapt the approach to the specific domain rather than applying it rigidly

**Avoid:**
- Applying the technique blindly without understanding the problem context
- Skipping validation steps to save time -- errors compound quickly
- Over-engineering the solution beyond what the task requires
- Ignoring the paper's stated limitations and applying it outside its scope

## Error Handling

- **Technique doesn't apply**: If the problem doesn't match Drift-Bench's assumptions, fall back to general-purpose reasoning and explain why
- **Partial results**: If some steps succeed but others fail, present the partial results with clear indication of what's missing
- **Conflicting information**: When sources disagree, present both sides with evidence and let the user decide
- **Performance issues**: If the approach is too slow, simplify by reducing the number of decomposition steps

## Limitations

- This skill encodes the *methodology* from the paper, not a trained model -- results depend on Claude's general capabilities
- The paper was evaluated on specific benchmarks; real-world tasks may differ significantly
- For tasks clearly outside the paper's scope, prefer general-purpose approaches

## Reference

**[Drift-Bench: Diagnosing Cooperative Breakdowns in LLM Agents under Input Faults via Multi-Turn Interaction](https://arxiv.org/abs/2602.02455v1)**
Key finding: Experiments show substantial performance drops under these faults, with clarification effectiveness varying across user personas and fault types.
Look for: methodology section, experimental setup, and ablation studies for tuning guidance.