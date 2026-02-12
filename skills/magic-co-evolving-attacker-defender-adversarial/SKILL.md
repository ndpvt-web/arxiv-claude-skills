---
name: "magic-co-evolving-attacker-defender-adversarial"
description: "Ensuring robust safety alignment is crucial for Large Language Models (LLMs), yet existing defenses often lag behind evolving adversarial attacks due to their \textbf{reliance on static, pre-collec... Implements the MAGIC approach. Use for: agent-framework, prompt-engineering. Triggers: 'orchestrate...', 'build a pipeline...', 'optimize this prompt...', 'improve the prompt for...'"
---

# MAGIC: A Co-Evolving Attacker-Defender Adversarial Game for Robust LLM Safety

This skill implements the approach described in *MAGIC: A Co-Evolving Attacker-Defender Adversarial Game for Robust LLM Safety*. In this paper, we introduce \textbf{MAGIC}, a novel multi-turn multi-agent reinforcement learning framework that formulates LLM safety alignment as an adversarial asymmetric game.

**Paper:** [https://arxiv.org/abs/2602.01539v2](https://arxiv.org/abs/2602.01539v2) | **Category:** cs.AI | **Published:** 2026-02-02
**Code:** [https://github.com/BattleWen/MAGIC.](https://github.com/BattleWen/MAGIC.)

## When to Use

- When orchestrating multiple steps or agents to solve a complex problem
- When designing or optimizing prompts for better AI performance
- When facing the challenge described in the paper: ensuring robust safety alignment is crucial for large language models (llms), yet existing defenses often lag behind evolving adversarial attacks due to their \textbf{reliance on static, pre-collected data distributions}.

## Core Technique

**The Problem:** Ensuring robust safety alignment is crucial for Large Language Models (LLMs), yet existing defenses often lag behind evolving adversarial attacks due to their \textbf{reliance on static, pre-collected data distributions}.

In this paper, we introduce \textbf{MAGIC}, a novel multi-turn multi-agent reinforcement learning framework that formulates LLM safety alignment as an adversarial asymmetric game.

Remarkably, we observe that the attacker, endowed with initial reasoning ability, evolves \textbf{novel, previously unseen combinatorial strategies} through iterative RL training, underscoring our method's substantial potential.

**Key Results:** Extensive experiments validate our framework's effectiveness, demonstrating superior defense success rates without compromising the helpfulness of the model.

## Step-by-Step Workflow

1. Analyze the task to determine if decomposition into subtasks provides value
2. Apply the MAGIC decomposition strategy to break the task into independent units
3. Design the agent topology: determine which subtasks can run in parallel vs sequentially
4. Define clear input/output contracts for each subtask (what goes in, what comes out)
5. Execute subtasks with appropriate error handling -- if one fails, others should still complete
6. Apply the paper's aggregation method to combine partial results into a unified answer
7. Validate the combined result for consistency and completeness
8. Present the final output with provenance tracking (which step produced what)

## Examples

**Example 1: Applying MAGIC**

```
User: Help me apply the MAGIC approach to my problem

Approach:
1. Understand the specific problem context and constraints
2. Map the problem to MAGIC's framework
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
- Read the full problem description before applying MAGIC
- Start with the simplest application of the technique and add complexity incrementally
- Validate intermediate results at each step of the workflow
- Adapt the approach to the specific domain rather than applying it rigidly

**Avoid:**
- Applying the technique blindly without understanding the problem context
- Skipping validation steps to save time -- errors compound quickly
- Over-engineering the solution beyond what the task requires
- Ignoring the paper's stated limitations and applying it outside its scope

## Error Handling

- **Technique doesn't apply**: If the problem doesn't match MAGIC's assumptions, fall back to general-purpose reasoning and explain why
- **Partial results**: If some steps succeed but others fail, present the partial results with clear indication of what's missing
- **Conflicting information**: When sources disagree, present both sides with evidence and let the user decide
- **Performance issues**: If the approach is too slow, simplify by reducing the number of decomposition steps

## Limitations

- This skill encodes the *methodology* from the paper, not a trained model -- results depend on Claude's general capabilities
- The paper was evaluated on specific benchmarks; real-world tasks may differ significantly
- The original problem context: ensuring robust safety alignment is crucial for large language models (llms), yet existing defenses often lag behind evolving adversarial attacks due to their \textbf{reliance on static, pre-collected data distributions}
- For tasks clearly outside the paper's scope, prefer general-purpose approaches

## Reference

**[MAGIC: A Co-Evolving Attacker-Defender Adversarial Game for Robust LLM Safety](https://arxiv.org/abs/2602.01539v2)**
Key finding: Extensive experiments validate our framework's effectiveness, demonstrating superior defense success rates without compromising the helpfulness of the model.
Implementation: [https://github.com/BattleWen/MAGIC.](https://github.com/BattleWen/MAGIC.)
Look for: methodology section, experimental setup, and ablation studies for tuning guidance.