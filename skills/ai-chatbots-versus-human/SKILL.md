---
name: "ai-chatbots-versus-human"
description: "Background: Empathy is widely recognized for improving patient outcomes, including reduced pain and anxiety and improved satisfaction, and its absence can cause harm Implements the AI chatbots versus human healthcare professionals approach. Use for: search-retrieval, database-query. Triggers: 'search for...', 'find information about...', 'write a SQL query...', 'query the database for...'"
---

# AI chatbots versus human healthcare professionals: a systematic review and meta-analysis of empathy in patient care

This skill implements the approach described in *AI chatbots versus human healthcare professionals: a systematic review and meta-analysis of empathy in patient care*. Background: Empathy is widely recognized for improving patient outcomes, including reduced pain and anxiety and improved satisfaction, and its absence can cause harm

**Paper:** [https://arxiv.org/abs/2602.05628v1](https://arxiv.org/abs/2602.05628v1) | **Category:** cs.HC | **Published:** 2026-02-05

## When to Use

- When searching, retrieving, and synthesizing information from multiple sources
- When writing or optimizing database queries

## Core Technique

Background: Empathy is widely recognized for improving patient outcomes, including reduced pain and anxiety and improved satisfaction, and its absence can cause harm. Meanwhile, use of artificial intelligence (AI)-based chatbots in healthcare is rapidly expanding, with one in five general practitioners using generative AI to assist with tasks such as writing letters. Some studies suggest AI chatbots can outperform human healthcare professionals (HCPs) in empathy, though findings are mixed and lack synthesis.   Sources of data: We searched multiple databases for studies comparing AI chatbots using large language models with human HCPs on empathy measures. We assessed risk of bias with ROBINS-I and synthesized findings using random-effects meta-analysis where feasible, whilst avoiding double counting.   Areas of agreement: We identified 15 studies (2023-2024). Thirteen studies reported statistically significantly higher empathy ratings for AI, with only two studies situated in dermatology favouring human responses. Of the 15 studies, 13 provided extractable data and were suitable for pooling. Meta-analysis of those 13 studies, all utilising ChatGPT-3.5/4, showed a standardized mean difference of 0.87 (95% CI, 0.54-1.20) favouring AI (P < .00001), roughly equivalent to a two-point increase on a 10-point scale.   Areas of controversy: Studies relied on text-based assessments that overlook non-verbal cues and evaluated empathy through proxy raters.   Growing points: Our findings indicate that, in text-only scenarios, AI chatbots are frequently perceived as more empathic than human HCPs.   Areas timely for developing research: Future research should validate these findings with direct patient evaluations and assess whether emerging voice-enabled AI systems can deliver similar empathic advantages.

**Key Results:** Background: Empathy is widely recognized for improving patient outcomes, including reduced pain and anxiety and improved satisfaction, and its absence can cause harm.

## Step-by-Step Workflow

1. Analyze the user's query to identify the core information need and any constraints
2. Decompose the query into 2-4 specific sub-questions that can be independently searched
3. Apply the AI chatbots versus human healthcare professionals approach: formulate multiple search strategies per sub-question
4. Execute searches across available sources (codebase, documentation, web, databases)
5. Rank results by relevance using the paper's scoring criteria: authority, recency, and semantic match
6. Cross-reference findings across sources to identify consensus and conflicts
7. Synthesize results into a structured answer with inline citations
8. Highlight confidence levels for each claim and flag any information gaps

## Examples

**Example 1: Multi-source information synthesis**

```
User: Research how to implement ai chatbots versus human healthcare professionals in my project

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
- Read the full problem description before applying AI chatbots versus human healthcare professionals
- Start with the simplest application of the technique and add complexity incrementally
- Validate intermediate results at each step of the workflow
- Adapt the approach to the specific domain rather than applying it rigidly

**Avoid:**
- Applying the technique blindly without understanding the problem context
- Skipping validation steps to save time -- errors compound quickly
- Over-engineering the solution beyond what the task requires
- Ignoring the paper's stated limitations and applying it outside its scope

## Error Handling

- **Technique doesn't apply**: If the problem doesn't match AI chatbots versus human healthcare professionals's assumptions, fall back to general-purpose reasoning and explain why
- **Partial results**: If some steps succeed but others fail, present the partial results with clear indication of what's missing
- **Conflicting information**: When sources disagree, present both sides with evidence and let the user decide
- **Performance issues**: If the approach is too slow, simplify by reducing the number of decomposition steps

## Limitations

- This skill encodes the *methodology* from the paper, not a trained model -- results depend on Claude's general capabilities
- The paper was evaluated on specific benchmarks; real-world tasks may differ significantly
- For tasks clearly outside the paper's scope, prefer general-purpose approaches

## Reference

**[AI chatbots versus human healthcare professionals: a systematic review and meta-analysis of empathy in patient care](https://arxiv.org/abs/2602.05628v1)**
Key finding: Background: Empathy is widely recognized for improving patient outcomes, including reduced pain and anxiety and improved satisfaction, and its absence can cause harm.
Look for: methodology section, experimental setup, and ablation studies for tuning guidance.