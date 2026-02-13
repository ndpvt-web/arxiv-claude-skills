---
name: "optimizing-small-sample-experience-learning-llm-ba"
description: "Implement the ExperienceWeaver hierarchical experience-learning framework to improve text quality from small feedback sets. Distills noisy corrections into structured Tips and Strategies, then injects them into a multi-agent detection-revision-critique pipeline. Use when: 'improve text from few examples', 'learn revision patterns from feedback', 'build experience-based text correction', 'distill feedback into reusable rules', 'small-sample text improvement pipeline', 'create agentic revision system from examples'."
---

# ExperienceWeaver: Small-Sample Experience Learning for Text Improvement

This skill enables Claude to implement the ExperienceWeaver framework from arXiv:2602.00740 — a hierarchical system that converts a small set of before/after correction examples (as few as 50-100) into structured, reusable revision knowledge. Rather than retrieving similar past examples (RAG) or fine-tuning a model, ExperienceWeaver distills feedback into two layers of actionable knowledge: error-specific **Tips** (concrete fixes for particular mistake types) and phase-level **Strategies** (high-level revision principles). These are injected into a three-agent pipeline (Detection → Revision → Self-Critique) that learns *how* to revise, not just *what* to revise.

## When to Use

- When the user has a small set of corrected text pairs (10-200 examples) and wants to build an automated improvement system from them
- When building a text-correction pipeline that needs to generalize from limited feedback rather than relying on large training data
- When the user wants to extract reusable revision rules from editor feedback, code review comments, or QA annotations
- When constructing a multi-agent revision workflow where each agent handles detection, correction, or validation
- When improving domain-specific text (medical reports, legal documents, technical docs) where expert corrections are scarce and expensive
- When RAG-based approaches return superficial fixes that miss the reasoning behind corrections

## Key Technique

**Experience distillation over example retrieval.** Standard RAG retrieves similar past corrections at inference time, but this gives the model *what* was changed without explaining *why*. ExperienceWeaver instead pre-processes all available feedback through two distillation stages. First, an **Experience Abstraction** step converts raw correction pairs into structured experience units — each one capturing the error pattern, the fix applied, and the underlying reasoning. Second, an **Experience Combination** step uses hierarchical tree-merging (grouping similar experiences in batches of ~4, then merging upward) to consolidate overlapping insights while preserving concrete details. The output is ~10 distilled experiences per error category, each 100-300 words.

**Two-layer knowledge injection.** The distilled experiences are further refined into two layers. **Tips** are error-specific: 5-8 concrete instructions per error type (50-100 words each), covering what's wrong and how to fix it. **Strategies** are phase-specific: 2-4 high-level directives (~50 words each) that guide overall behavior during detection, revision, or self-critique. This separation lets the system inject precisely relevant knowledge — Tips when a specific error is detected, Strategies to guide overall approach — without overwhelming the context window.

**Agentic pipeline with experience injection.** Three specialized agents coordinate via an orchestrator: the **Error Detection Agent** identifies and classifies mistakes, the **Revision Agent** generates improvements using injected Tips for detected errors, and the **Self-Critique Agent** evaluates output quality and triggers re-revision if the score falls below a threshold (up to 3 iterations). Each agent receives the relevant Strategies for its phase plus error-specific Tips matched to the current input.

## Step-by-Step Workflow

1. **Collect correction pairs.** Gather before/after text pairs with optional feedback annotations. Even 50 pairs suffice. Structure each pair as `{original, corrected, feedback_notes}`. If only corrected versions exist, generate diffs programmatically to identify what changed.

2. **Classify error types from the pairs.** Diff each pair to extract individual corrections. Cluster corrections into error categories (e.g., "missing units", "ambiguous reference", "incorrect terminology", "structural inconsistency"). Use an LLM to propose category names if manual labeling is impractical. Track frequency of each error type across the dataset.

3. **Abstract raw feedback into experience units.** For each correction pair, prompt an LLM to produce a structured experience unit: `{error_type, error_pattern, correction_applied, reasoning, generalization}`. The generalization field captures *why* the fix works, not just *what* was done. Aim for 100-300 words per unit.

4. **Combine experiences via hierarchical merging.** Group experience units by error type. Within each group, merge in batches of 4 (the NG hyperparameter): prompt the LLM to consolidate overlapping insights, remove redundancy, and preserve concrete details. Repeat merging on the merged outputs until you have ~10 distilled experiences per error category.

5. **Distill Tips from combined experiences.** For each error type, prompt the LLM to synthesize the distilled experiences into 5-8 actionable Tips (50-100 words each). Each Tip should specify: the error pattern to watch for, the correction approach, and a brief rationale. Apply a frequency threshold (τe): only generate Tips for errors appearing in >10% of your training pairs.

6. **Synthesize Strategies from Tips.** Aggregate all Tips by pipeline phase (detection, revision, self-critique). Prompt the LLM to produce 2-4 high-level Strategies per phase (~50 words each) that capture the overall revision style and priorities. Strategies answer "what kind of reviser should I be?" rather than "how do I fix error X?"

7. **Build the three-agent pipeline.** Implement three agents with distinct system prompts:
   - **Detection Agent**: receives input text + detection Strategies; outputs a list of `{error_type, location, description}` entries.
   - **Revision Agent**: receives input text + detected errors + relevant Tips (max 5 per invocation, controlled by τt) + revision Strategies; outputs corrected text.
   - **Self-Critique Agent**: receives original + revised text + critique Strategies; scores the revision (0-1) and lists remaining issues. If score < 0.6, loop back to the Revision Agent (max 3 iterations).

8. **Wire the orchestrator.** Build a coordinator that sequences the agents: Detection → Revision → Self-Critique → (conditional loop). The orchestrator selects which Tips to inject based on the Detection Agent's error classifications. Use a ReAct-style loop where the orchestrator decides whether to re-invoke revision based on critique output.

9. **Validate on held-out pairs.** Reserve 20-50% of your correction pairs for testing. Run the pipeline on original texts and compare outputs against known corrections. Measure improvement using domain-appropriate metrics (BLEU, ROUGE, expert scoring, or error-category accuracy).

10. **Iterate on the experience library.** When new correction pairs arrive, run them through steps 2-6 to update Tips and Strategies incrementally. The experience library grows without retraining any model.

## Concrete Examples

**Example 1: Building a clinical report improvement system**

```
User: I have 80 pairs of draft and corrected radiology reports. Build me a
system that can automatically improve new draft reports using patterns from
these corrections.

Approach:
1. Parse the 80 pairs, diff each to extract individual corrections.
   Cluster into error types: "missing laterality" (23 occurrences),
   "vague measurement" (18), "inconsistent comparison" (15),
   "abbreviation misuse" (12), "incomplete impression" (9), others (3).

2. Abstract each correction into an experience unit:
   {
     "error_type": "missing laterality",
     "error_pattern": "Findings described without specifying left/right",
     "correction_applied": "Added 'left' before 'lower lobe'",
     "reasoning": "Laterality is required for all paired structures to
                   prevent wrong-site procedures",
     "generalization": "Any finding involving paired anatomical structures
                        must explicitly state laterality"
   }

3. Merge the 23 "missing laterality" units in groups of 4, producing
   ~6 consolidated experiences, then distill into 6 Tips:
   - Tip 1: "Check all references to lungs, kidneys, adrenals, and
     extremities for explicit left/right designation..."
   - Tip 2: "When a finding is midline, state 'midline' explicitly..."

4. Synthesize phase Strategies:
   - Detection Strategy: "Prioritize laterality, measurement precision,
     and impression completeness — the three highest-frequency error
     categories in this institution's corrections."
   - Revision Strategy: "Preserve the original report's structure and
     voice. Insert corrections minimally rather than rewriting sections."

5. Deploy pipeline: Detection Agent flags 3 errors in a new draft →
   Revision Agent injects relevant Tips and produces corrected report →
   Self-Critique Agent scores 0.78, accepts the revision.

Output (for a new draft report):
  Original: "Opacity in the lower lobe. No pleural effusion."
  Detected: [{error: "missing laterality", location: "lower lobe"}]
  Revised:  "Opacity in the right lower lobe. No pleural effusion."
```

**Example 2: Code review feedback distillation**

```
User: I have 60 PR review comments with before/after code snippets. Help me
build an automated code improvement agent that applies these review patterns.

Approach:
1. Parse review comments into correction pairs. Classify error types:
   "missing error handling" (15), "inconsistent naming" (12),
   "unnecessary complexity" (10), "missing type annotations" (8),
   "security: unsanitized input" (6), other (9).

2. Abstract each into experience units. Example:
   {
     "error_type": "missing error handling",
     "error_pattern": "async function calls without try/catch or .catch()",
     "correction_applied": "Wrapped fetch call in try/catch with specific
                            error type checking",
     "reasoning": "Unhandled promise rejections crash the process in
                   production; specific error types enable proper fallback",
     "generalization": "All external I/O (network, filesystem, database)
                        needs explicit error handling with typed catches"
   }

3. Distill into Tips:
   - Tip 1: "Wrap all fetch/axios calls in try/catch. Catch specific
     error types (NetworkError, TimeoutError) before a generic fallback."
   - Tip 2: "Database queries must handle connection errors separately
     from query errors — connection errors should trigger retry logic."

4. Strategies:
   - Detection: "Focus on I/O boundaries and type safety first."
   - Revision: "Add error handling without restructuring surrounding code.
     Match the existing project patterns for error types and logging."

5. Pipeline processes new code: detects 2 issues → revises with Tips →
   self-critique scores 0.85, accepts.

Output:
  Detected: ["missing error handling at line 42 (fetch without catch)",
             "inconsistent naming at line 15 (mixedCase vs snake_case)"]
  Revised: [code with targeted fixes applied]
```

**Example 3: Technical documentation improvement**

```
User: Our tech writers corrected 40 API docs. Extract patterns and build
a system to improve new drafts automatically.

Approach:
1. Diff the 40 pairs. Error types: "missing parameter description" (14),
   "ambiguous return type" (10), "no error code documentation" (8),
   "inconsistent formatting" (5), "missing example" (3).

2. Distill experiences, merge, produce Tips and Strategies.
   - Tip (missing parameter): "Every parameter must have: name, type,
     required/optional, default value if optional, and a one-sentence
     description. Check against the function signature."
   - Strategy (detection): "Validate completeness before style. Missing
     information is more harmful than imperfect wording."

3. Run pipeline on new API doc drafts. Detection Agent finds gaps,
   Revision Agent fills them using Tips, Self-Critique validates.
```

## Best Practices

- **Do:** Keep Tips concrete and error-specific (50-100 words). Vague tips like "write better" are useless. Each Tip should specify what to look for and exactly how to fix it.
- **Do:** Set a frequency threshold for Tip generation. Errors appearing in fewer than 10% of examples may be noise rather than patterns. The τe hyperparameter controls this.
- **Do:** Limit injected Tips to 5 per revision invocation (τt=5). More Tips degrade performance by overwhelming the context window with competing instructions.
- **Do:** Keep the Self-Critique loop to a maximum of 3 iterations. Diminishing returns set in quickly, and infinite loops waste tokens.
- **Avoid:** Injecting all Tips regardless of detected error type. The Detection Agent's classifications should gate which Tips the Revision Agent receives.
- **Avoid:** Skipping the hierarchical merging step. Raw experience units contain too much redundancy and noise to use directly as Tips. The tree-merging (groups of 4) is essential for consolidation.
- **Avoid:** Using this framework when you have thousands of examples — at that scale, fine-tuning or standard RAG may be more cost-effective. ExperienceWeaver's advantage is specifically in the 10-200 example range.

## Error Handling

- **Too few examples (<10):** The distillation step may not find enough patterns to generalize. Fall back to direct few-shot prompting with the available pairs until more data is collected.
- **Overly heterogeneous error types:** If each example has a unique error type with no repetition, the merging step produces no consolidation. Lower the frequency threshold or use broader error categories.
- **Self-Critique loop never converges:** If the critique agent consistently scores below 0.6 after 3 iterations, the Tips may be contradictory or the task may be beyond single-pass correction. Log the failing cases and use them to refine Tips manually.
- **Context window overflow:** With many error types detected simultaneously, injecting Tips for all of them can exceed context limits. The τt parameter (max 5 Tips) prevents this — prioritize Tips for the most frequent or severe error types.
- **Domain drift:** Tips distilled from one document type (e.g., radiology reports) won't transfer to another (e.g., discharge summaries). Maintain separate experience libraries per domain.

## Limitations

- Requires at least some corrected examples — cannot bootstrap from zero feedback. If no correction pairs exist, this framework cannot help.
- The quality ceiling is set by the quality of the input corrections. If expert feedback is inconsistent or wrong, the distilled Tips will inherit those flaws.
- Works best for text domains with recurring, classifiable error patterns. Creative writing or highly variable content may not exhibit the regularity needed for Tip distillation.
- The three-agent pipeline adds latency (3+ LLM calls per input). For high-throughput batch processing, consider distilling the Tips and Strategies into a single comprehensive system prompt instead of running the full agentic loop.
- Performance gains diminish on text that is already high quality. The framework's strength is correcting systematic errors, not polishing near-perfect prose.

## Reference

**Paper:** [ExperienceWeaver: Optimizing Small-sample Experience Learning for LLM-based Clinical Text Improvement](https://arxiv.org/abs/2602.00740v1) — Xiao et al., 2026. Focus on Section 3 (framework architecture), Section 3.2 (hierarchical experience distillation), and Section 3.3 (agentic pipeline design) for implementation details. Table 2 shows performance gains across datasets; Table 4 details the hyperparameter sensitivity analysis for NG, τe, and τt.