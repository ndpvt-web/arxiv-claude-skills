---
name: "c-mop-integrating-momentum-boundary-aware"
description: "Optimize LLM system prompts iteratively using boundary-aware contrastive sampling and momentum-guided clustering from the C-MOP framework. Use when: 'optimize this prompt', 'improve my system prompt', 'evolve prompt for better accuracy', 'automatic prompt tuning', 'prompt optimization with examples', 'refine prompt using test cases'."
---

# C-MOP: Momentum and Boundary-Aware Prompt Evolution

This skill enables Claude to systematically optimize LLM prompts using the C-MOP (Cluster-based Momentum Optimized Prompting) framework. Instead of ad-hoc prompt tweaking, C-MOP applies two structured mechanisms: **Boundary-Aware Contrastive Sampling (BACS)** to identify the most informative success/failure cases, and **Momentum-Guided Semantic Clustering (MGSC)** to accumulate stable optimization signals across iterations. The result is a disciplined prompt evolution process that avoids the noisy, contradictory edits typical of naive prompt refinement.

## When to Use

- When the user has a system prompt and a set of test cases (input/expected-output pairs) and wants to maximize accuracy
- When iterative prompt editing keeps flip-flopping between fixes (solving one case breaks another)
- When the user asks to "optimize", "evolve", or "tune" a prompt against evaluation data
- When a prompt works on easy cases but fails on edge cases or ambiguous inputs
- When the user wants to systematically improve a prompt for a classification, extraction, or reasoning task
- When migrating a prompt from a stronger model to a weaker one and needing to recover lost performance

## Key Technique

**The core problem** with naive prompt optimization is conflicting signals: fixing failure case A introduces a change that breaks previously-passing case B. Each iteration oscillates rather than converges. C-MOP solves this with two mechanisms.

**Boundary-Aware Contrastive Sampling (BACS)** categorizes evaluation examples into three groups after each prompt trial: *Anchors* (cases the prompt consistently gets right -- these define "what's working"), *Hard Negatives* (cases the prompt consistently gets wrong -- these define "what's broken"), and *Boundary Pairs* (cases near the decision boundary that sometimes pass, sometimes fail -- these reveal where the prompt is ambiguous). Instead of feeding the optimizer a random mix of successes and failures, BACS selects a structured triplet of anchor + hard negative + boundary case. This gives the optimization signal maximum contrast: "keep doing X (anchor), stop doing Y (hard negative), and clarify Z (boundary)."

**Momentum-Guided Semantic Clustering (MGSC)** maintains a history of suggested prompt edits ("gradients") across iterations and clusters them semantically using embeddings. Edits that recur across multiple iterations get amplified (they represent persistent issues), while one-off suggestions decay via a temporal weight factor (`momentum = alpha * previous_momentum + (1 - alpha) * new_gradient`, with alpha typically 0.8-0.9). This filters out noise and surfaces the stable consensus about what the prompt actually needs.

## Step-by-Step Workflow

1. **Collect the prompt and evaluation set.** Obtain the user's current system prompt and a set of at least 10-20 test cases with inputs and expected outputs. More cases (50+) yield better boundary detection. Parse them into a structured list: `[{input, expected_output, category (optional)}]`.

2. **Run baseline evaluation.** Execute the current prompt against all test cases. Record each result as pass/fail with the actual output. Compute baseline accuracy. This is iteration 0.

3. **Classify examples using BACS tripartite sampling.** Sort results into three bins:
   - **Anchors**: Cases that pass reliably (score = 1.0). Sample 2-3 representative anchors that cover different input patterns.
   - **Hard Negatives**: Cases that fail clearly (score = 0.0). Sample 2-3 that represent distinct failure modes.
   - **Boundary Pairs**: Cases with partial or inconsistent results (if running multiple trials) or cases where the output is "almost right." Sample 1-2.

4. **Generate structured optimization gradient.** Present the LLM optimizer with the current prompt plus the BACS triplet in this format:
   ```
   Current prompt: [prompt text]

   ANCHOR (working correctly):
   Input: [anchor input] -> Expected: [expected] -> Got: [correct output]
   Analysis: The prompt handles this well because [reason].

   HARD NEGATIVE (failing):
   Input: [hard neg input] -> Expected: [expected] -> Got: [wrong output]
   Analysis: The prompt fails here because [diagnosis].

   BOUNDARY CASE (ambiguous):
   Input: [boundary input] -> Expected: [expected] -> Got: [partial/wrong output]
   Analysis: The prompt is unclear about [specific ambiguity].

   Task: Generate 2-3 specific, minimal edits to the prompt that fix the hard negative
   and boundary case WITHOUT breaking the anchor case pattern.
   ```

5. **Apply momentum to candidate edits.** Maintain a running list of all suggested edits from previous iterations. Cluster semantically similar suggestions (e.g., "add output format specification" and "clarify expected response structure" are the same theme). Weight each cluster by recurrence count and recency:
   - `weight = count * decay^(current_iteration - last_seen_iteration)` where decay = 0.85
   - Prioritize edit themes with highest weight -- these represent persistent consensus.

6. **Generate candidate prompts.** Produce 3-4 prompt variants by applying the top-weighted edit themes. Each variant should make minimal, targeted changes. Keep a "beam" of the top candidates (beam_size = 4 is the paper's default).

7. **Evaluate candidates on a fresh minibatch.** Score each candidate prompt against a sample of test cases (the paper uses 256 samples per eval round). Rank by accuracy.

8. **Select and iterate.** Keep the top-performing candidates (beam search). Return to step 3 with the new best prompt. Repeat for 10-20 rounds or until accuracy plateaus (less than 0.5% improvement over 3 consecutive rounds).

9. **Final validation.** Run the best prompt against the full held-out test set. Compare against baseline. Report the improvement and the specific changes made.

10. **Document the evolution trace.** Output a summary showing: baseline accuracy, per-round accuracy, the key edit themes that persisted (high momentum), and the final optimized prompt.

## Concrete Examples

**Example 1: Optimizing a sentiment classification prompt**

User: "I have a prompt for classifying customer reviews as positive/negative/neutral but it's only 72% accurate on my test set. Help me optimize it."

Approach:
1. Collect the current prompt and test cases. Run baseline: 72% accuracy.
2. BACS classification after baseline run:
   - Anchors: Clear positive ("Love this product!") and clear negative ("Terrible, broke immediately") -- prompt handles extremes fine.
   - Hard Negatives: Sarcastic reviews ("Oh great, another update that breaks everything" classified as positive) and mixed reviews ("Good quality but overpriced" classified as positive instead of neutral).
   - Boundary: Conditional reviews ("Would be great if it worked as advertised" -- hovers between negative and neutral).
3. Generate gradient: "The prompt lacks sarcasm detection guidance and has no criteria for neutral classification."
4. After 3 iterations, momentum clustering reveals persistent theme: "neutral category is under-specified" (weight: 2.55). One-off suggestion "add emoji handling" decays away (weight: 0.32).
5. Apply high-momentum edit: Add explicit neutral criteria to the prompt.

Output after 8 rounds:
```
Baseline accuracy: 72%
Final accuracy: 84%
Key persistent edits (high momentum):
  1. Added explicit neutral classification criteria (+6% lift)
  2. Added sarcasm detection instruction (+4% lift)
  3. Specified tie-breaking rule for mixed sentiment (+2% lift)
Decayed (noise) suggestions filtered out:
  - "Add emoji interpretation" (appeared once, iteration 3)
  - "Use chain-of-thought" (appeared once, iteration 5, no improvement)
```

**Example 2: Improving a code review prompt for a smaller model**

User: "My code review prompt works with GPT-4 but degrades badly with a 7B model. Can you optimize it for the smaller model?"

Approach:
1. Run the current prompt against 30 code review test cases on the 7B model. Baseline: 45% of reviews match expert quality.
2. BACS sampling reveals:
   - Anchors: Simple single-issue reviews (unused variable, missing return) -- the 7B model handles atomic issues.
   - Hard Negatives: Multi-issue code snippets where the model only finds 1 of 3 issues, or hallucinates non-existent bugs.
   - Boundary: Cases where the model identifies the right issue but gives a wrong fix suggestion.
3. Iteration 1 gradient: "Break complex reviews into single-issue passes" and "Add explicit instruction to not invent issues."
4. Iteration 2-4: Momentum accumulates on "structured output format" theme (the model performs better with constrained output).
5. By iteration 10, the optimized prompt includes: numbered issue format, one-issue-per-pass instruction, "only report issues you can quote from the code" guard rail.

Output:
```
Baseline (7B model): 45%
Optimized (7B model): 71%
Optimized prompt changes:
  - Added structured output template (issue/location/fix format)
  - Added "only cite issues present in the provided code" instruction
  - Added "review for one category at a time" sequential approach
  - Removed abstract instructions the 7B model couldn't follow
```

**Example 3: Evolving a data extraction prompt with contradictory failures**

User: "My prompt for extracting dates from legal documents keeps oscillating -- when I fix US date formats, European formats break."

Approach:
1. Run baseline on 50 legal document excerpts. Accuracy: 68%.
2. BACS reveals the core conflict:
   - Anchors: Unambiguous dates ("January 15, 2024", "2024-01-15").
   - Hard Negatives: "01/02/2024" misclassified based on whichever format was last fixed.
   - Boundary: "the first Monday of March" -- sometimes extracted, sometimes missed.
3. Without momentum, iterations 1-4 oscillate: fix US format (+5%), break EU format (-4%), fix EU format (+3%), break US format (-3%).
4. MGSC momentum clustering after 4 iterations identifies the persistent signal: "format disambiguation requires document-level context clue detection" (weight: 3.2). The oscillating "prefer US format" / "prefer EU format" suggestions cancel out (net weight: ~0).
5. High-momentum edit applied: Add instruction to first identify the document's jurisdiction/origin, then apply the corresponding date format convention.

Output after 12 rounds:
```
Baseline: 68%
After naive iteration (no momentum): 70% (oscillating)
After C-MOP optimization: 82%
Key insight surfaced by momentum: Format disambiguation requires
jurisdiction detection as a prerequisite step, not a per-case rule.
```

## Best Practices

- **Do:** Start with at least 20 diverse test cases. BACS needs enough examples to identify meaningful anchors, hard negatives, and boundary cases. Fewer than 10 cases produces unreliable classifications.
- **Do:** Track the full edit history across iterations and cluster by semantic similarity. The momentum signal only works if you accumulate suggestions over 5+ rounds.
- **Do:** Make minimal, isolated edits per candidate. If you change 5 things at once, you cannot attribute which change helped or hurt. The paper uses beam_size=4 candidates with small targeted changes each.
- **Do:** Use a held-out test set for final validation that was never used during the optimization rounds to detect overfitting to the training examples.
- **Avoid:** Treating all failures equally. BACS specifically distinguishes hard negatives (consistent failures) from boundary cases (inconsistent results). Mixing them produces muddled optimization signals.
- **Avoid:** Acting on one-off optimization suggestions. If an edit idea only appears in a single iteration, it likely addresses noise rather than a systematic prompt weakness. Let the momentum mechanism filter it.
- **Avoid:** Running too few iterations. The paper uses 20 rounds. Stopping at 3-4 rounds means the momentum mechanism hasn't had time to separate signal from noise.

## Error Handling

- **Too few test cases for BACS:** If fewer than 10 test cases are available, skip the tripartite classification and instead use a simple pass/fail split. Note to the user that results will be less stable.
- **No boundary cases found:** If all cases either clearly pass or clearly fail, the prompt's decision boundary is sharp. Focus optimization on hard negatives only and skip boundary pair analysis.
- **Momentum pool is empty (first iteration):** On iteration 1, there's no history to cluster. Generate edits based on BACS sampling alone. Momentum kicks in from iteration 2 onward.
- **All candidates perform equally:** If beam candidates show <1% spread, the prompt may be at a local optimum. Introduce a higher-temperature mutation (more aggressive rewrites) to escape the plateau, then resume normal optimization.
- **Contradictory persistent themes:** If momentum surfaces two high-weight clusters that conflict (e.g., "be more specific" vs. "be more general"), this signals the prompt is being asked to handle fundamentally different subtasks. Recommend splitting into two specialized prompts.

## Limitations

- **Requires evaluation data:** C-MOP cannot optimize a prompt without input/output test cases. For open-ended creative tasks with no measurable ground truth, this approach does not apply.
- **Computationally intensive:** Each iteration requires running the prompt against a batch of examples. With 20 rounds and 256 samples per round, that's 5000+ LLM calls for a single optimization run. For expensive API models, cost adds up.
- **Marginal gains on already-strong prompts:** The paper shows 1.5-3.5% average improvement over strong baselines. If a prompt is already at 95%+ accuracy, C-MOP may yield diminishing returns.
- **Assumes stable task distribution:** If the test cases don't represent real production inputs well, the optimized prompt may overfit to the evaluation set.
- **Single-prompt scope:** C-MOP optimizes one system prompt at a time. For multi-turn conversation flows or tool-using agents, the prompt interactions are more complex than what this framework directly addresses.

## Reference

**Paper:** [C-MOP: Integrating Momentum and Boundary-Aware Clustering for Enhanced Prompt Evolution](https://arxiv.org/abs/2602.10874v1) (Yan et al., 2026). Look for: Algorithm 1 (BACS tripartite sampling), Algorithm 2 (MGSC momentum update with temporal decay), and Table 2 (benchmark results showing 3B model surpassing 70B via optimized prompts).

**Code:** [github.com/huawei-noah/noah-research/tree/master/C-MOP](https://github.com/huawei-noah/noah-research/tree/master/C-MOP) -- reference implementation with configurable rounds (default 20), beam_size (default 4), and UCB bandit evaluation.