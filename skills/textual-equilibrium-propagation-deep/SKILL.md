---
name: "textual-equilibrium-propagation-deep"
description: "Optimize deep compound AI pipelines using Textual Equilibrium Propagation (TEP) -- a two-phase local learning framework that replaces brittle global textual backpropagation with equilibrium-seeking local critics and nudged contrastive updates. Use when: 'optimize my multi-step LLM pipeline', 'my agent chain is losing quality at depth', 'improve my compound AI system', 'fix degrading feedback in my prompt chain', 'textual gradient optimization', 'TEP for agentic workflows'."
---

# Textual Equilibrium Propagation for Deep Compound AI Systems

This skill enables Claude to design, optimize, and debug deep compound AI systems -- multi-module LLM pipelines where retrievers, generators, verifiers, and tools are chained together. It applies Textual Equilibrium Propagation (TEP), which replaces fragile global textual backpropagation (as in TextGrad) with a two-phase local optimization: a **free phase** where local critics refine each module's prompt until equilibrium, and a **nudged phase** where bounded task-level signals propagate forward to produce contrastive updates. TEP maintains stable performance as pipeline depth grows, unlike global methods that suffer exploding or vanishing textual gradients.

## When to Use

- When the user has a multi-step LLM pipeline (3+ chained modules) and wants to optimize prompts end-to-end
- When a compound AI system degrades in quality as more modules are added (depth-scaling failure)
- When textual feedback from a final evaluator becomes too long, generic, or unstable when propagated backward through many modules
- When building agentic workflows with retrieval -> reasoning -> code generation -> verification chains
- When the user asks to "optimize" or "improve" a multi-agent or multi-prompt pipeline
- When debugging why a deep prompt chain produces worse results than its individual components
- When designing a new compound AI system that needs to be optimizable from the start

## Key Technique

**The Problem with Global Textual Backpropagation:** Approaches like TextGrad compute a loss at the final output, then propagate textual feedback backward through every module. This fails at depth for two reasons: (1) **Exploding textual gradients** -- feedback message length grows exponentially as `B(g) >= c * gamma^k` (gamma > 1), overflowing context windows and amplifying evaluation biases; (2) **Vanishing textual gradients** -- specificity decays as `S(g) <= C * alpha^k` (alpha in (0,1)), where actionable corrections like "fix the date format in step 3" degrade into useless generic suggestions like "improve accuracy" after passing through several modules.

**TEP's Two-Phase Solution:** Instead of global backprop, TEP optimizes each module locally using two alternating phases. In the **free phase**, a local LLM critic evaluates each module's output on structured quality metrics (structural clarity, specification completeness, internal consistency, context integration, reasoning transparency, format compliance) and suggests improvements. The module iterates until the critic's scores stabilize (variance across 3 consecutive evaluations < 0.5) or a maximum iteration cap is reached. In the **nudged phase**, the system injects task-level ground truth or objective signals at the output, then runs the pipeline forward again with these clamped targets. Each module compares its free-phase output against its nudged-phase output to produce a **contrastive update** -- the textual diff between "what I produced on my own" and "what I should have produced given the correct answer." This contrastive signal is local, bounded, and does not degrade with depth.

**Why It Scales:** TEP's nudge strength parameter beta decays geometrically (beta <- beta * 0.9 per round), starting with strong coordination and transitioning to fine-tuning. Every update is validated: edits that reduce performance on a held-out set are rejected. This makes TEP monotonically non-degrading. On benchmarks, TEP improved HotpotQA F1 from 24.86 (TextGrad) to 48.72, and maintained stable token complexity at pipeline scale 5 while TextGrad's feedback ballooned from 2K to 32K+ tokens.

## Step-by-Step Workflow

1. **Map the pipeline graph.** Enumerate every module (node) in the compound AI system and its connections. Label each node with its role: retriever, generator, verifier, code-writer, tool-caller, etc. Identify the depth (longest path from input to output).

2. **Define local critic metrics for each node.** For every module, create a structured evaluation rubric with 6 task-independent dimensions (scored 1-5): Structural Clarity, Specification Completeness, Internal Consistency, Context Integration, Reasoning Transparency, Format Compliance. Add 2-3 task-dependent criteria specific to the module's role (e.g., "retrieval recall" for a retriever, "code correctness" for a code generator).

3. **Initialize module prompts.** Write an initial system prompt for each module. Keep prompts modular -- each module should have a clear input schema and output schema so that local optimization doesn't break inter-module contracts.

4. **Run the free phase per module.** For each module independently: (a) generate output from current prompt, (b) have the local critic score it on all metrics and produce actionable feedback, (c) revise the prompt incorporating the feedback, (d) repeat until score variance across 3 consecutive evaluations < 0.5, or until 20 iterations. Use temperature sampling uniformly in [0.3, 0.9] across iterations.

5. **Run the full pipeline forward (free-phase equilibrium).** Execute the entire pipeline end-to-end with the locally-optimized prompts. Record the output at every module -- these are the free-phase activations.

6. **Inject nudged signals.** Clamp the final output to the ground-truth or objective target. Run the pipeline forward again, but now each module receives its predecessor's nudged output instead of free output. Record these nudged-phase activations at every module.

7. **Compute contrastive updates.** For each module, compare its free-phase output to its nudged-phase output. Generate a textual diff: "In the free phase you produced X, but the correct pipeline produced Y. The key differences are [specific, bounded list]." Use this diff to revise the module's prompt with proximal edits only -- changes must be minimal and specific, not wholesale rewrites.

8. **Validate and accept/reject.** Test each proposed prompt edit against a small validation set. Only accept edits that do not reduce validation performance. This ensures monotonic non-degradation.

9. **Decay nudge strength and iterate.** Multiply beta by 0.9 and repeat from step 5. Early rounds make large structural corrections; later rounds fine-tune. Stop when validation performance plateaus across 3 consecutive rounds.

10. **Export optimized prompts.** Freeze the final prompts for each module. Document the critic rubrics alongside each prompt so the system can be re-optimized when requirements change.

## Concrete Examples

**Example 1: Optimizing a Multi-Hop QA Pipeline**

User: "I have a 4-module pipeline for answering complex questions: Query Decomposer -> Retriever -> Evidence Synthesizer -> Answer Generator. Quality drops badly when I chain all 4 together. Help me optimize it."

Approach:
1. Map the pipeline: 4 nodes, depth 4, linear chain.
2. Define local critics:
   - Query Decomposer: clarity of sub-questions, coverage of original query, no redundancy
   - Retriever: passage relevance, diversity, recall of key entities
   - Evidence Synthesizer: faithful summarization, no hallucinated claims, covers all sub-questions
   - Answer Generator: correctness, conciseness, cites evidence
3. Run free phase on each module independently using 10 sample questions, iterating prompts until critic scores stabilize.
4. Run full pipeline forward, record all intermediate outputs.
5. Clamp Answer Generator output to gold answers. Re-run pipeline forward with clamped targets flowing backward as context.
6. For each module, generate contrastive update:
   ```
   Evidence Synthesizer contrastive diff:
   Free phase: "The study was conducted in 2019 and found positive results."
   Nudged phase: "The 2019 RCT by Smith et al. (n=500) found 23% improvement (p<0.01)."
   Key gap: Free phase drops specific figures and citations. Add to prompt:
   "Always include author names, sample sizes, and statistical significance."
   ```
7. Validate each edit, accept only improvements, decay beta, repeat 5 rounds.

Output: Optimized prompts for all 4 modules with documented critic rubrics. Expected: significant F1 improvement at depth 4, stable feedback token counts.

**Example 2: Scaling a Code Generation Pipeline**

User: "I'm building: Problem Analyzer -> Code Generator -> Test Generator -> Code Refiner. At scale, the feedback from the refiner back to the analyzer becomes useless generic text. Fix this."

Approach:
1. Diagnose the failure mode: this is **vanishing textual gradient** -- feedback loses specificity over 4 hops.
2. Replace global backprop with TEP local critics:
   - Problem Analyzer critic: checks for edge case identification, input/output spec completeness, constraint enumeration
   - Code Generator critic: checks syntax validity, algorithm correctness, efficiency, style
   - Test Generator critic: checks coverage of edge cases, assertion quality, independence from implementation
   - Code Refiner critic: checks bug fixes, performance improvements, readability
3. Free phase: optimize each module's prompt until local critic scores stabilize.
4. Nudged phase: clamp Code Refiner output to known-correct solutions. Compare what each module produced freely vs. what it should have produced given the correct final code.
5. Contrastive update for Problem Analyzer:
   ```
   Free: "Handle integer inputs, return sorted list"
   Nudged: "Handle integer inputs including negatives and duplicates,
   return sorted list in O(n log n), raise ValueError on empty input"
   Edit: Add to prompt: "Explicitly enumerate: negative numbers, duplicates,
   empty inputs, type constraints, and complexity requirements."
   ```
6. Validate on 20 held-out problems, accept only non-degrading edits.

Output: Each module's prompt is optimized locally. Feedback stays bounded (under 500 tokens per module) regardless of pipeline depth.

**Example 3: Designing a TEP-Ready Agentic Workflow from Scratch**

User: "I want to build a research agent that searches papers, extracts claims, verifies them, and writes a summary. How should I structure it for optimization?"

Approach:
1. Design 4 modules with explicit input/output schemas:
   - Paper Searcher: query -> ranked paper list (title, abstract, relevance score)
   - Claim Extractor: paper text -> structured claims (claim, evidence, confidence)
   - Claim Verifier: claim + evidence -> verification verdict + reasoning
   - Summary Writer: verified claims -> coherent research summary
2. Build local critic rubrics for each module at design time:
   ```json
   {
     "module": "Claim Verifier",
     "task_independent": {
       "structural_clarity": "1-5: Is the verdict clearly separated from reasoning?",
       "internal_consistency": "1-5: Does the reasoning support the verdict?",
       "reasoning_transparency": "1-5: Are logical steps explicit?"
     },
     "task_dependent": {
       "evidence_grounding": "1-5: Does verdict rely only on provided evidence?",
       "uncertainty_calibration": "1-5: Are confidence levels appropriate?"
     }
   }
   ```
3. Initialize prompts with clear contracts between modules.
4. Collect 30 sample research questions, split 20/10 train/validation.
5. Run TEP optimization: free phase (local critic convergence), nudged phase (clamp to expert summaries), contrastive updates, validation gating, beta decay. Run 8 rounds.

Output: A 4-module pipeline with optimized prompts, documented critic rubrics, and stable performance at depth 4.

## Best Practices

- **Do:** Define explicit input/output schemas for every module before optimization. TEP's local critics need clear contracts to evaluate against.
- **Do:** Start with a generous iteration cap (20) for the free phase, then reduce once you observe typical convergence behavior (usually 5-8 iterations suffice).
- **Do:** Use the structured 6-dimension scoring rubric (Structural Clarity, Specification Completeness, Internal Consistency, Context Integration, Reasoning Transparency, Format Compliance) as your baseline -- it is task-agnostic and well-calibrated.
- **Do:** Decay beta geometrically (multiply by 0.9 each round) to transition from coarse structural fixes to fine-grained tuning.
- **Avoid:** Propagating raw final-output feedback backward through more than 2 modules -- this is exactly the failure mode TEP replaces.
- **Avoid:** Making large prompt rewrites in contrastive updates. Edits must be proximal: specific, bounded, and targeted at the exact gap identified in the free-vs-nudged comparison.
- **Avoid:** Skipping the validation gate. Every prompt edit must be tested against held-out examples. Accepting unvalidated edits breaks monotonic improvement guarantees.

## Error Handling

- **Free phase fails to converge:** If a module's critic scores oscillate without stabilizing after 20 iterations, the critic rubric is likely ambiguous or conflicting. Simplify the rubric to fewer, more orthogonal dimensions. Reduce temperature range to [0.3, 0.6] to decrease output variance.
- **Nudged phase produces nonsensical outputs:** The clamped target may be incompatible with the module's input format. Verify that the ground-truth signal can be meaningfully interpreted at each module's level of abstraction. For intermediate modules, derive intermediate targets from the final ground truth rather than clamping the raw answer.
- **Contrastive updates are empty (free = nudged):** The module is already performing optimally or the nudge signal isn't reaching it. Check that upstream modules are passing nudged outputs forward correctly. Consider clamping at intermediate modules, not just the final one.
- **Validation performance drops despite good contrastive diffs:** The edits are overfitting to the nudged examples. Increase validation set size or reduce edit specificity (make edits slightly more general).

## Limitations

- TEP requires ground-truth or high-quality objective targets for the nudged phase. It cannot optimize purely open-ended creative pipelines where "correct output" is undefined.
- The free phase's local critics add computational cost -- each module runs 5-20 evaluation iterations. For latency-sensitive applications, consider running free-phase optimization offline and deploying frozen prompts.
- TEP is designed for pipelines with 3+ modules. For 2-module systems, simple TextGrad-style feedback is usually sufficient and simpler.
- The technique assumes modules communicate via text. Pipelines with non-textual intermediate representations (embeddings, images) require adaptation at those boundaries.
- Local critics may have blind spots that only manifest at the system level. Periodic end-to-end evaluation is still necessary alongside local optimization.

## Reference

[Textual Equilibrium Propagation for Deep Compound AI Systems](https://arxiv.org/abs/2601.21064v2) -- Chen, Deng, Zou, Yu, Li (2026). Focus on Section 3 (the two-phase TEP algorithm), Section 4.1 (the structured critic rubric), and Figure 3 (depth-scaling comparison against TextGrad). Key result: TEP achieves 48.72 F1 on HotpotQA vs TextGrad's 24.86 at depth 4, with stable token complexity.