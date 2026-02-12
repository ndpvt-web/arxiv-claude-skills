---
name: "scaling-in-context-online-learning"
description: "Large language models (LLMs) achieve strong performance when all task-relevant information is available upfront, as in static prediction and instruction-following problems Implements the Scaling In-Context Online Learning Capability of LLMs via Cross-Episode Meta-RL approach. Use for: search-retrieval, agent-framework, prompt-engineering, security. Triggers: 'search for...', 'find information about...', 'orchestrate...', 'build a pipeline...', 'optimize this prompt...', 'improve the prompt for...'"
---

# Scaling In-Context Online Learning Capability of LLMs via Cross-Episode Meta-RL

This skill implements the approach described in *Scaling In-Context Online Learning Capability of LLMs via Cross-Episode Meta-RL*. We introduce ORBIT, a multi-task, multi-episode meta-reinforcement learning framework that trains LLMs to learn from interaction in context.

**Paper:** [https://arxiv.org/abs/2602.04089v1](https://arxiv.org/abs/2602.04089v1) | **Category:** cs.AI | **Published:** 2026-02-03
**Code:** [https://github.com/XiaofengLin7/ORBIT.](https://github.com/XiaofengLin7/ORBIT.)

## When to Use

- When searching, retrieving, and synthesizing information from multiple sources
- When orchestrating multiple steps or agents to solve a complex problem
- When designing or optimizing prompts for better AI performance
- When auditing code or systems for security vulnerabilities
- When facing the challenge described in the paper: large language models (llms) achieve strong performance when all task-relevant information is available upfront, as in static prediction and instruction-following problems.

## Core Technique

**The Problem:** Large language models (LLMs) achieve strong performance when all task-relevant information is available upfront, as in static prediction and instruction-following problems.

We introduce ORBIT, a multi-task, multi-episode meta-reinforcement learning framework that trains LLMs to learn from interaction in context.

**Key Results:** Large language models (LLMs) achieve strong performance when all task-relevant information is available upfront, as in static prediction and instruction-following problems.

## Step-by-Step Workflow

1. Analyze the user's query to identify the core information need and any constraints
2. Decompose the query into 2-4 specific sub-questions that can be independently searched
3. Apply the Scaling In-Context Online Learning Capability of LLMs via Cross-Episode Meta-RL approach: formulate multiple search strategies per sub-question
4. Execute searches across available sources (codebase, documentation, web, databases)
5. Rank results by relevance using the paper's scoring criteria: authority, recency, and semantic match
6. Cross-reference findings across sources to identify consensus and conflicts
7. Synthesize results into a structured answer with inline citations
8. Highlight confidence levels for each claim and flag any information gaps

## Examples

**Example 1: Multi-source information synthesis**

```
User: Research how to implement scaling in-context online learning capability of llms via cross-episode meta-rl in my project

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
- Read the full problem description before applying Scaling In-Context Online Learning Capability of LLMs via Cross-Episode Meta-RL
- Start with the simplest application of the technique and add complexity incrementally
- Validate intermediate results at each step of the workflow
- Adapt the approach to the specific domain rather than applying it rigidly

**Avoid:**
- Applying the technique blindly without understanding the problem context
- Skipping validation steps to save time -- errors compound quickly
- Over-engineering the solution beyond what the task requires
- Ignoring the paper's stated limitations and applying it outside its scope

## Error Handling

- **Technique doesn't apply**: If the problem doesn't match Scaling In-Context Online Learning Capability of LLMs via Cross-Episode Meta-RL's assumptions, fall back to general-purpose reasoning and explain why
- **Partial results**: If some steps succeed but others fail, present the partial results with clear indication of what's missing
- **Conflicting information**: When sources disagree, present both sides with evidence and let the user decide
- **Performance issues**: If the approach is too slow, simplify by reducing the number of decomposition steps

## Limitations

- This skill encodes the *methodology* from the paper, not a trained model -- results depend on Claude's general capabilities
- The paper was evaluated on specific benchmarks; real-world tasks may differ significantly
- The original problem context: large language models (llms) achieve strong performance when all task-relevant information is available upfront, as in static prediction and instruction-following problems
- For tasks clearly outside the paper's scope, prefer general-purpose approaches

## Reference

**[Scaling In-Context Online Learning Capability of LLMs via Cross-Episode Meta-RL](https://arxiv.org/abs/2602.04089v1)**
Key finding: Large language models (LLMs) achieve strong performance when all task-relevant information is available upfront, as in static prediction and instruction-following problems.
Implementation: [https://github.com/XiaofengLin7/ORBIT.](https://github.com/XiaofengLin7/ORBIT.)
Look for: methodology section, experimental setup, and ablation studies for tuning guidance.