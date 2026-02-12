---
name: textual-equilibrium-propagation-deep
description: >
  Optimize deep multi-step AI pipelines using Textual Equilibrium Propagation (TEP) — a two-phase
  local-then-nudge strategy that avoids gradient explosion/vanishing in long prompt chains.
  Use when: "optimize my multi-agent pipeline", "fix my prompt chain that degrades at depth",
  "improve my compound AI system", "TEP optimization", "local critic refinement for agents",
  "scale my LLM workflow without losing quality".
---

# Textual Equilibrium Propagation for Deep Compound AI Systems

This skill enables Claude to design, optimize, and debug deep compound AI systems — multi-step LLM pipelines where each stage feeds into the next (retrieval, reasoning, code generation, verification, etc.). It applies Textual Equilibrium Propagation (TEP), a technique from Chen et al. (ICLR 2026) that replaces fragile global feedback chains with a two-phase approach: first, each module self-improves via a local LLM critic until stable (free phase), then receives minimal task-aligned nudges propagated forward (nudged phase). This prevents the twin failures of deep prompt optimization — feedback that explodes in length or vanishes in specificity — and scales to 5x+ pipeline depth where global methods like TextGrad collapse.

## When to Use

- When the user has a multi-step LLM pipeline (3+ stages) and wants to optimize prompts across stages without manual tuning
- When a compound AI system degrades in quality as depth increases (e.g., a 5-stage agent chain performs worse than a 3-stage one)
- When global feedback propagation (TextGrad-style) produces bloated or vague optimization signals
- When building retrieval-augmented generation pipelines, multi-agent tool-use workflows, or code generation pipelines with analysis/generation/testing/refinement stages
- When the user wants to add self-critique loops to individual agents in a pipeline without destabilizing the whole system
- When optimizing agentic workflows where each agent's prompt needs tuning but agents are black-box LLM calls

## Key Technique

**The Problem with Global Textual Backpropagation:** Methods like TextGrad treat LLM pipelines like neural networks and backpropagate textual feedback from the final output through every upstream module. This works for shallow pipelines (2-3 steps) but fails at depth. Feedback token count grows exponentially (~2.2x per additional stage), and when compressed to fit context windows, the feedback loses specificity — downstream update success rates drop from 36% to 5% at 5x scale.

**TEP's Two-Phase Solution:** Inspired by Equilibrium Propagation from energy-based models, TEP decouples optimization into local and global concerns. In the **free phase**, each node in the pipeline gets its own LLM critic that iteratively evaluates and refines the node's prompt/output against six quality dimensions (structural clarity, completeness, consistency, context integration, reasoning transparency, format compliance) until scores stabilize — this is "equilibrium." In the **nudged phase**, small task-aligned edits are applied to each node's prompt using the actual task objective (e.g., final answer accuracy), propagated via forward signaling rather than backward chains. A nudge strength parameter beta (annealed by 0.9x per iteration) bounds how much each edit can change, preventing oscillation. The result: stable token costs across depth, consistent 33-37% update success rates even at 5x scale, and accuracy gains that grow with depth.

**Why It Works in Practice:** Each module optimizes itself locally first, reaching a solid baseline without depending on noisy global signals. The nudged phase then makes surgical adjustments to align local optima with the global objective. This means you can add pipeline depth without degrading optimization quality — the opposite of TextGrad's behavior.

## Step-by-Step Workflow

1. **Map the pipeline as a computation graph.** Identify every LLM call, tool invocation, and data transform as a node. Draw directed edges showing data flow. Label each node as stochastic (LLM call with tunable prompt) or deterministic (tool, database query, formatter). Only stochastic nodes get optimized.

2. **Define the task-level objective.** Specify a measurable evaluation function for the pipeline's final output — accuracy, F1, pass@1, MRR, or a rubric-based score. This is the global signal used in the nudged phase. Make it concrete and automatable.

3. **Attach a local critic to each stochastic node.** For each LLM node, create a critic prompt that evaluates the node's output on six dimensions scored 1-5: structural clarity (weight 0.20), specification completeness (0.20), internal consistency (0.15), context integration (0.15), reasoning transparency (0.15), and format compliance (0.15). The critic outputs a JSON with scores, actionable feedback, and an overall weighted score.

4. **Run the free phase: local self-refinement to equilibrium.** For each stochastic node independently (parallelizable), loop: generate output, have the critic evaluate it, apply suggested improvements to the prompt, regenerate. Stop when score variance across 3 consecutive evaluations falls below 0.5, when feedback becomes stylistic rather than functional, or after 20 iterations max. This produces locally-optimized prompts.

5. **Evaluate the full pipeline end-to-end.** Run the pipeline with all locally-optimized prompts on a sample of task instances. Compute the task-level objective. This is the baseline that the nudged phase will improve upon.

6. **Run the nudged phase: task-aligned forward nudges.** For each stochastic node, generate a minimal prompt edit ("nudge") that aligns the node's behavior with the global task objective. Start with nudge strength beta > 0 and anneal by 0.9x each outer iteration. The nudge is a specific, bounded text edit — not a rewrite. Apply it, re-run the pipeline forward, and check if the global objective improves.

7. **Iterate nudge-then-evaluate until convergence.** Repeat the nudged phase for up to 40 iterations or until the global objective plateaus. Each iteration: apply nudges, run forward, measure, anneal beta. Track which nodes' nudges improved vs. degraded performance.

8. **Validate on held-out data.** Test the final optimized prompts on examples not used during optimization. Compare against the pre-optimization baseline and any global-optimization baseline (TextGrad, DSPy) if available.

9. **Export the optimized prompt set.** Save each node's final prompt as a versioned artifact. Document which nodes changed most and what the critics focused on — this is diagnostic gold for future pipeline debugging.

## Concrete Examples

**Example 1: Optimizing a Multi-Hop QA Pipeline**

User: "I have a 4-stage pipeline for HotpotQA: query decomposition -> sub-question retrieval -> evidence synthesis -> answer generation. Accuracy drops when I add more decomposition depth. Help me optimize it with TEP."

Approach:
1. Map the 4 nodes: Decomposer (stochastic), Retriever (deterministic), Synthesizer (stochastic), Answerer (stochastic)
2. Task objective: F1 score on final answers
3. Attach critics to Decomposer, Synthesizer, Answerer. Example critic prompt for Decomposer:

```
You are a critic evaluating a query decomposition module.
Given the original question and the generated sub-questions, score on:
- Structural Clarity (1-5): Are sub-questions logically ordered?
- Completeness (1-5): Do sub-questions cover all information needed?
- Consistency (1-5): Do sub-questions avoid redundancy/contradiction?
- Context Integration (1-5): Does decomposition use entity/relation cues from the question?
- Reasoning Transparency (1-5): Is the decomposition rationale clear?
- Format Compliance (1-5): Are sub-questions properly formatted for the retriever?

Output JSON: {"scores": {...}, "actionable_feedback": "...", "overall": <float>}
```

4. Free phase results (after ~12 iterations each):
   - Decomposer equilibrium: overall 4.1 (improved sub-question specificity)
   - Synthesizer equilibrium: overall 3.8 (improved evidence citation)
   - Answerer equilibrium: overall 4.3 (improved answer formatting)
5. Nudged phase: Forward-signal nudge to Decomposer — "Ensure sub-questions include temporal constraints when the original question involves dates" (beta=0.7, annealed over 15 iterations)
6. Result: F1 improves from 44.9 (DSPy baseline) to 48.7

**Example 2: Code Generation Pipeline Optimization**

User: "My code generation system has: problem analysis -> code gen -> test gen -> code refinement. It works at depth 1x but fails at 3x scale. Optimize it."

Approach:
1. Map nodes: Analyzer (stochastic), Generator (stochastic), TestWriter (stochastic), Refiner (stochastic)
2. Task objective: pass@1 on BigCodeBench
3. Attach critics. Example for Generator:

```
Evaluate this generated code against the problem analysis:
- Structural Clarity: Is the code well-organized with clear function boundaries?
- Completeness: Does it implement all requirements from the analysis?
- Consistency: Are variable names, error handling patterns consistent?
- Context Integration: Does it use the data structures suggested by the analysis?
- Reasoning Transparency: Are algorithmic choices justified in comments?
- Format Compliance: Does it follow the required output format?
```

4. Free phase: Each node self-refines independently. Generator reaches equilibrium at iteration 8 (score 4.0). Refiner needs 18 iterations (score 3.6 — harder to self-evaluate).
5. Nudged phase: Key nudge to Refiner — "When test failures indicate edge cases, modify only the specific failing branch rather than restructuring" (beta=0.5). Key nudge to TestWriter — "Generate boundary-value tests before happy-path tests" (beta=0.6).
6. Result: pass@1 improves from 35.7 (TextGrad) to 39.0, with token cost 41% lower at 2x scale

**Example 3: Retrieval-Augmented Pipeline with Tool Use**

User: "I need to optimize a research assistant pipeline: query understanding -> web search -> document ranking -> summarization -> fact verification -> response generation."

Approach:
1. Map 6 nodes: QueryParser (stochastic), WebSearch (deterministic), Ranker (stochastic), Summarizer (stochastic), Verifier (stochastic), Responder (stochastic)
2. Task objective: MRR on STARK-PRIME benchmark
3. Free phase on 4 stochastic nodes in parallel. Ranker critic focuses on relevance ordering; Verifier critic focuses on claim-evidence alignment.
4. Nudged phase: Forward signal from MRR evaluation nudges QueryParser to include domain-specific qualifiers, and Ranker to weight recency more heavily.
5. Result: MRR improves from 41.4 (DSPy) to 42.7 while keeping token cost flat across the 6-stage depth

## Best Practices

- **Do:** Run the free phase for all nodes in parallel — they are independent by design. This is the primary efficiency advantage over sequential global backpropagation.
- **Do:** Use the six-dimension scoring rubric consistently across all critics. Customize weights per node type (e.g., format compliance matters more for a code generator than a summarizer).
- **Do:** Start nudge strength beta at 0.5-0.8 and anneal by 0.9x. Too high causes oscillation; too low causes stagnation.
- **Do:** Sample temperature uniformly from U(0.3, 0.9) during optimization to avoid overfitting to a single decoding mode.
- **Avoid:** Skipping the free phase and jumping straight to nudges. The free phase establishes a stable local optimum that nudges can meaningfully adjust. Without it, HotpotQA F1 drops by 11.9 points.
- **Avoid:** Using global feedback chains for pipelines deeper than 3 stages. Token growth is exponential (~2.2x per stage) and update success rates collapse to 5% at 5x depth.
- **Avoid:** Making nudges that rewrite prompts wholesale. Nudges must be proximal — small, targeted edits. If a nudge changes more than 15-20% of a prompt, reduce beta.

## Error Handling

- **Critic scores oscillate without converging:** Lower the temperature for the critic LLM call (use 0.3 instead of sampling from U(0.3, 0.9)) and increase the variance threshold from 0.5 to 1.0. If still oscillating after 20 iterations, accept the current state as approximate equilibrium.
- **Nudges degrade global performance:** Roll back the nudge, reduce beta by 50%, and retry. If three consecutive nudges all degrade, the node may already be at its local-global optimum — skip it.
- **Token budget exceeded during free phase:** Reduce max iterations from 20 to 10 and use a smaller/faster critic model (e.g., Claude Haiku instead of Opus). The free phase is the most token-intensive component.
- **Pipeline has deterministic nodes that bottleneck quality:** TEP only optimizes stochastic (LLM) nodes. If a tool or database query is the bottleneck, refactor the pipeline to add a stochastic preprocessing node before the deterministic one (e.g., a query formatter before a search API).
- **Nudge phase shows no improvement:** Verify the task-level objective is measurable and differentiable at the prompt level. Vague objectives like "be better" produce vague nudges. Use specific metrics (F1, pass@1, MRR).

## Limitations

- TEP requires a measurable end-to-end evaluation metric. If the final output can only be judged subjectively, the nudged phase lacks a clear signal.
- The free phase adds latency per node (up to 20 critic iterations). For latency-sensitive applications, cap iterations aggressively (5-8) and accept approximate equilibrium.
- TEP optimizes prompts, not model weights. If a pipeline's bottleneck is model capability (the LLM simply cannot do the task), prompt optimization will plateau.
- Works best at depth 3+. For 2-stage pipelines, simpler approaches (direct prompt engineering, TextGrad) may be equally effective with less overhead.
- The six-dimension rubric is general-purpose. Highly specialized domains (medical, legal) may need custom scoring dimensions for the critic.

## Reference

Chen, M., Deng, W., Zou, J., Yu, H., & Li, X. (2026). *Textual Equilibrium Propagation for Deep Compound AI Systems.* ICLR 2026. [arXiv:2601.21064v2](https://arxiv.org/abs/2601.21064v2)

Key sections: Section 3 for the formal TEP algorithm, Section 4.1 for depth-scaling failure analysis of TextGrad, Table 2 for benchmark comparisons, Appendix B for full pseudocode, and Appendix C for critic prompt templates.