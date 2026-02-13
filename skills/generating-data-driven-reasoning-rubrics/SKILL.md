---
name: "generating-data-driven-reasoning-rubrics"
description: "Build granular error taxonomies from incorrect reasoning traces, then use those rubrics to detect errors in LLM outputs across technical domains. Use when asked to: 'build a rubric for evaluating code solutions', 'create an error taxonomy for math reasoning', 'grade reasoning traces for correctness', 'build a reward function for domain-specific tasks', 'classify errors in chain-of-thought outputs', 'evaluate LLM reasoning without gold labels'."
---

# Generating Data-Driven Reasoning Rubrics

This skill enables Claude to automatically construct granular reasoning-error taxonomies ("rubrics") from collections of incorrect reasoning traces, then apply those rubrics to detect errors in unseen outputs. Based on Sanders et al. (2026), the technique replaces vague "is this correct?" prompting with structured, domain-specific error classification. The rubric acts as a checklist of known failure modes -- each with a keyword for fast retrieval and verification criteria for precise detection -- yielding up to +45% accuracy improvement over generic LLM-as-judge baselines on difficult technical domains.

## When to Use

- When the user wants to evaluate correctness of LLM-generated reasoning (code solutions, math proofs, engineering analyses) and simple answer-checking is insufficient.
- When building a reward function or scoring pipeline for reinforcement learning on domain-specific tasks.
- When the user has a set of known-incorrect outputs and wants to systematically catalog the error patterns.
- When grading student or model outputs on technical problems where mistakes are subtle and domain-specific.
- When the user needs error detection that works with only 20% of the gold labels that a verifiable reward approach would require.
- When evaluating long chain-of-thought traces where errors are buried in extended reasoning.

## Key Technique

**The core insight:** Instead of asking an LLM "is this reasoning correct?" (which fails on long traces and expert domains), you first mine a corpus of known-bad traces to build a structured error taxonomy, then use that taxonomy as a retrieval-augmented checklist during evaluation. This converts an open-ended judgment into a structured classification task, which LLMs handle much more reliably.

**Rubric construction** works by compressing each incorrect reasoning trace into a summary of its logical steps (removing exploratory dead-ends), then prompting an LLM to identify the specific error that caused the wrong answer. Each extracted error becomes a rubric item with three fields: (1) a concise error description under 25 words, (2) a keyword or short phrase for retrieval, and (3) verification details explaining how to detect the error in a trace. Related keywords are grouped to deduplicate, reducing rubric size by roughly 50%.

**Error classification** uses a two-stage pipeline. Stage 1 (high recall): tag a new trace with all potentially relevant keywords from the rubric. Stage 2 (high precision): retrieve only the rubric items matching those keywords and check each against the compressed trace. If any rubric item applies, the trace is classified as incorrect. This retrieve-then-verify approach keeps the context window focused and avoids overwhelming the judge with hundreds of rubric items at once.

## Step-by-Step Workflow

1. **Collect incorrect traces.** Gather 50-200+ reasoning outputs known to be wrong (via ground-truth answers, test suites, or manual labels). More traces yield a more complete rubric; 200 is the practical minimum for technical domains.

2. **Compress each trace.** For each incorrect trace, prompt an LLM to summarize only the logical steps that influence the final answer, stripping exploratory tangents and self-corrections that were abandoned. Output a numbered list of reasoning steps.

3. **Extract error descriptions.** For each compressed trace, prompt the LLM: "Given this reasoning and the known-incorrect answer, identify the specific error(s) that caused the wrong result." Collect the response as a candidate rubric item.

4. **Structure each rubric item.** For every extracted error, produce three fields:
   - `description`: A plain-language error description in under 25 words.
   - `keyword`: A 1-4 word retrieval tag (e.g., "off-by-one indexing", "unit conversion", "sign error").
   - `verification`: 1-3 sentences explaining how to confirm whether this error exists in a given trace.

5. **Deduplicate and group keywords.** Prompt the LLM to cluster semantically similar keywords (e.g., "boundary condition" and "edge case handling" might merge). This typically reduces rubric size by ~50%. Skip this step for narrow problem domains where errors are already distinct.

6. **Validate rubric coverage.** Run the rubric against a held-out validation set (20% of your labeled data). Check that specificity (fraction of incorrect traces caught) exceeds 75%. If coverage is low, add more source traces and repeat steps 2-5.

7. **Classify new traces -- Stage 1 (Keyword Tagging).** Given a new reasoning trace to evaluate, compress it (step 2), then prompt: "Which of these keywords are potentially relevant to this trace?" providing the full keyword list. Collect all matched keywords.

8. **Classify new traces -- Stage 2 (Verification).** Retrieve the full rubric items for matched keywords only. Prompt: "For each rubric item below, determine whether the described error is present in this trace. Cite the specific step where the error occurs, or state that it does not apply." If any item matches, classify the trace as incorrect.

9. **Aggregate into a reward signal.** For RL or ranking pipelines, convert the classification into a binary reward (1 = no errors found, 0 = error detected). Optionally, output the matched rubric items as structured feedback for the model.

10. **Iterate the rubric.** Periodically re-run rubric construction on newly collected incorrect traces to capture novel error patterns not in the original taxonomy.

## Concrete Examples

**Example 1: Building a rubric for code solution evaluation**

```
User: I have 150 failed solutions to LeetCode-style problems. Build me a rubric
      to evaluate future solutions from my coding model.

Approach:
1. Compress each failed solution's reasoning trace to its key logical steps
   (algorithm choice, data structure selection, loop logic, edge case handling).
2. For each, extract the root cause error. Example outputs:
   - description: "Fails to handle empty input array"
     keyword: "empty input"
     verification: "Check if the solution tests for len(arr)==0 before accessing indices."
   - description: "Uses O(n^2) approach exceeding time limit on large inputs"
     keyword: "time complexity"
     verification: "Identify nested loops over the input; check if n > 10^4 makes this infeasible."
   - description: "Off-by-one error in binary search bounds"
     keyword: "off-by-one"
     verification: "Check loop condition (< vs <=) and mid calculation for fencepost errors."
3. Group related keywords: merge "index out of bounds" with "off-by-one".
4. Final rubric: ~120 items covering algorithm errors, edge cases, complexity
   issues, and language-specific pitfalls.

Output (rubric excerpt):
| # | Keyword            | Description                              | Verification                                      |
|---|--------------------|------------------------------------------|----------------------------------------------------|
| 1 | empty input        | Fails to handle empty input array        | Check for guard clause on len(arr)==0              |
| 2 | off-by-one         | Incorrect loop boundary in binary search | Verify < vs <= in while condition; check mid calc  |
| 3 | integer overflow   | Intermediate sum exceeds 32-bit range    | Look for unchecked addition of large values        |
| 4 | time complexity    | Quadratic approach on large input        | Count nested loops; verify n constraints           |
```

**Example 2: Evaluating a math reasoning trace using an existing rubric**

```
User: Here's a model's chain-of-thought for a calculus problem. Use the math
      rubric to check if its reasoning is correct.

Approach:
1. Compress the trace: extract the 8 key steps (problem setup, substitution,
   integration by parts, limit evaluation, simplification, final answer).
2. Stage 1 -- keyword tagging: match against rubric keywords.
   Matched: "sign error", "integration bounds", "substitution variable".
3. Stage 2 -- verification:
   - "sign error" rubric item: "Check whether negative signs are preserved
     through algebraic manipulation." -> Step 4 drops a negative sign when
     moving -cos(x) outside the integral. ERROR FOUND.
   - "integration bounds" rubric item: "Verify bounds are correctly
     transformed after variable substitution." -> Bounds correctly updated. OK.
   - "substitution variable" rubric item: "Confirm du/dx is correctly
     computed." -> Derivative is correct. OK.
4. Classification: INCORRECT (sign error at step 4).

Output:
  Classification: INCORRECT
  Matched error: "sign error" (rubric item #17)
  Location: Step 4 -- the factor of -1 from cos(x) derivative is dropped
  Confidence: HIGH (verified against specific algebraic step)
```

**Example 3: Building a reward function for RL training on chemistry problems**

```
User: I want to train a model on organic chemistry synthesis problems but only
      have gold answers for 50 out of 250 problems. Build me a reward function.

Approach:
1. Use the 50 gold-labeled problems to generate ~200 incorrect traces
   (sample 4 model outputs per problem, keep those that disagree with gold).
2. Build rubric from incorrect traces:
   - "reagent incompatibility": Proposes reagents that react with each other.
   - "stereochemistry neglect": Ignores chiral centers during substitution.
   - "protecting group omission": Fails to protect reactive functional groups.
   - ... (~150 items total after deduplication)
3. Validate: on held-out 10 problems (40 traces), rubric catches 85% of
   incorrect traces with 90% precision.
4. Deploy as reward function: for each RL rollout, compress the trace,
   run two-stage classification, return reward = 1 if no rubric items match.
5. The remaining 200 unlabeled problems can now be used for RL training
   with the rubric-based reward, no gold labels needed.

Output:
  Rubric: 153 items across 12 error categories
  Validation specificity: 85%
  Validation balanced accuracy: 87%
  Ready for use as binary reward function in RL pipeline.
```

## Best Practices

- **Do:** Compress traces before error extraction. Raw chain-of-thought traces contain exploratory dead-ends that confuse error analysis. Always summarize to the steps that influenced the final answer.
- **Do:** Use the three-field rubric format (description, keyword, verification). The keyword enables fast retrieval; the verification criteria prevent false positives during classification.
- **Do:** Validate rubric coverage on held-out data before deploying. Aim for >75% specificity and >80% balanced accuracy.
- **Do:** Rebuild rubrics periodically as the model improves -- old error patterns get fixed and new ones emerge.
- **Avoid:** Sending the entire rubric (200+ items) to the judge in one prompt. The two-stage retrieve-then-verify pipeline is essential; without it, the LLM is overwhelmed and accuracy drops.
- **Avoid:** Building rubrics from fewer than 50 incorrect traces. The taxonomy will be too sparse to catch the diversity of real errors.
- **Avoid:** Applying rubrics cross-domain without regeneration. A math rubric won't catch coding errors. Rubrics are domain-specific by design.

## Error Handling

- **Low specificity (<60%):** The rubric is missing common error patterns. Collect more incorrect traces from the underperforming category and regenerate.
- **High false-positive rate:** Verification criteria are too vague. Tighten the `verification` field with more specific detection instructions (cite exact patterns to look for, not general descriptions).
- **Keyword explosion (>400 items before dedup):** The source traces span too many problem types. Segment by sub-domain (e.g., algebra vs. geometry) and build separate rubrics.
- **Compression loses critical steps:** The compressor LLM is over-summarizing. Adjust the compression prompt to retain all computational steps, not just high-level strategy.
- **Stage 1 matches too many keywords:** Lower the threshold or instruct the tagger to select only the top-5 most relevant keywords per trace.

## Limitations

- Rubrics are only as good as the source traces. If your incorrect-trace corpus doesn't contain a particular failure mode, the rubric won't catch it.
- The technique works best on domains with structured, identifiable errors (math, code, engineering). It is less effective on subjective or creative tasks where "correctness" is ambiguous.
- Rubric construction requires an LLM capable of understanding domain-specific errors. For highly specialized fields (e.g., quantum chemistry), the LLM's own knowledge limits rubric quality.
- The two-stage pipeline adds latency compared to a single-call judge. For real-time applications, consider caching keyword matches.
- Rubrics degrade over time as the model being evaluated improves and develops new error patterns not in the original taxonomy.

## Reference

Sanders, K., Weir, N., Chaudhary, S., Bostrom, K., & Rangwala, H. (2026). *Generating Data-Driven Reasoning Rubrics for Domain-Adaptive Reward Modeling.* arXiv:2602.06795v1. Key sections: Section 3 (rubric construction pipeline), Section 4 (two-stage classification), Section 5.2 (RL reward modeling results showing +45% over baseline judges).