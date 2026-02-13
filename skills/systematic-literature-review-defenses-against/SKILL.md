---
name: "systematic-literature-review-defenses-against"
description: "The rapid advancement and widespread adoption of generative artificial intelligence (GenAI) and large language models (LLMs) has been accompanied by the emergence of new security vulnerabilities an... Implements the A Systematic Literature Review on LLM Defenses Against Prompt Injection and Jailbreaking approach. Use for: data-processing, search-retrieval, prompt-engineering, security. Triggers: 'parse this data...', 'extract from...', 'search for...', 'find information about...', 'optimize this prompt...', 'improve the prompt for...'"
---

# A Systematic Literature Review on LLM Defenses Against Prompt Injection and Jailbreaking: Expanding NIST Taxonomy

This skill implements the approach described in *A Systematic Literature Review on LLM Defenses Against Prompt Injection and Jailbreaking: Expanding NIST Taxonomy*. Second, we propose an extension to NIST taxonomy by introducing additional categories of defenses.

**Paper:** [https://arxiv.org/abs/2601.22240v1](https://arxiv.org/abs/2601.22240v1) | **Category:** cs.CR | **Published:** 2026-01-29

## When to Use

- When extracting, cleaning, or transforming data from various formats
- When searching, retrieving, and synthesizing information from multiple sources
- When designing or optimizing prompts for better AI performance
- When auditing code or systems for security vulnerabilities
- When facing the challenge described in the paper: the rapid advancement and widespread adoption of generative artificial intelligence (genai) and large language models (llms) has been accompanied by the emergence of new security vulnerabilities and challenges, such as jailbreaking and other prompt injection attacks.

## Core Technique

**The Problem:** The rapid advancement and widespread adoption of generative artificial intelligence (GenAI) and large language models (LLMs) has been accompanied by the emergence of new security vulnerabilities and challenges, such as jailbreaking and other prompt injection attacks.

Second, we propose an extension to NIST taxonomy by introducing additional categories of defenses.

## Step-by-Step Workflow

1. Analyze the task to determine if decomposition into subtasks provides value
2. Apply the A Systematic Literature Review on LLM Defenses Against Prompt Injection and Jailbreaking decomposition strategy to break the task into independent units
3. Design the agent topology: determine which subtasks can run in parallel vs sequentially
4. Define clear input/output contracts for each subtask (what goes in, what comes out)
5. Execute subtasks with appropriate error handling -- if one fails, others should still complete
6. Apply the paper's aggregation method to combine partial results into a unified answer
7. Validate the combined result for consistency and completeness
8. Present the final output with provenance tracking (which step produced what)

## Examples

**Example 1: Applying A Systematic Literature Review on LLM Defenses Against Prompt Injection and Jailbreaking**

```
User: Help me apply the A Systematic Literature Review on LLM Defenses Against Prompt Injection and Jailbreaking approach to my problem

Approach:
1. Understand the specific problem context and constraints
2. Map the problem to A Systematic Literature Review on LLM Defenses Against Prompt Injection and Jailbreaking's framework
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
- Read the full problem description before applying A Systematic Literature Review on LLM Defenses Against Prompt Injection and Jailbreaking
- Start with the simplest application of the technique and add complexity incrementally
- Validate intermediate results at each step of the workflow
- Adapt the approach to the specific domain rather than applying it rigidly

**Avoid:**
- Applying the technique blindly without understanding the problem context
- Skipping validation steps to save time -- errors compound quickly
- Over-engineering the solution beyond what the task requires
- Ignoring the paper's stated limitations and applying it outside its scope

## Error Handling

- **Technique doesn't apply**: If the problem doesn't match A Systematic Literature Review on LLM Defenses Against Prompt Injection and Jailbreaking's assumptions, fall back to general-purpose reasoning and explain why
- **Partial results**: If some steps succeed but others fail, present the partial results with clear indication of what's missing
- **Conflicting information**: When sources disagree, present both sides with evidence and let the user decide
- **Performance issues**: If the approach is too slow, simplify by reducing the number of decomposition steps

## Limitations

- This skill encodes the *methodology* from the paper, not a trained model -- results depend on Claude's general capabilities
- The paper was evaluated on specific benchmarks; real-world tasks may differ significantly
- The original problem context: the rapid advancement and widespread adoption of generative artificial intelligence (genai) and large language models (llms) has been accompanied by the emergence of new security vulnerabilities and challenges, such as jailbreaking and other prompt injection attacks
- For tasks clearly outside the paper's scope, prefer general-purpose approaches

## Reference

**[A Systematic Literature Review on LLM Defenses Against Prompt Injection and Jailbreaking: Expanding NIST Taxonomy](https://arxiv.org/abs/2601.22240v1)**
Look for: methodology section, experimental setup, and ablation studies for tuning guidance.