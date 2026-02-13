---
name: "discovering-process-outcome-credit-multi-step"
description: >
  Apply Step-wise Marginal Information Gain (MIG) credit assignment to multi-step reasoning tasks.
  Evaluates each reasoning step by its marginal contribution toward the correct answer rather than
  by position or final outcome alone. Use this skill when asked to: "evaluate my chain-of-thought",
  "score each reasoning step", "find where my logic goes wrong", "credit assign my solution steps",
  "debug my multi-step reasoning", "identify which steps actually matter in this derivation".
---

# Process-Outcome Credit Assignment for Multi-Step Reasoning

This skill enables Claude to evaluate, score, and improve multi-step reasoning chains by applying the Marginal Information Gain (MIG) framework from Wang et al. (2026). Instead of judging reasoning only by its final answer (outcome credit) or by step position (naive process credit), this approach measures each step's *intrinsic semantic contribution* — how much closer it moves toward the correct solution compared to the best progress achieved so far. This produces precise, position-independent credit assignment that identifies pivotal breakthroughs, flags redundant steps, and pinpoints where reasoning derails.

## When to Use

- When a user shares a multi-step derivation (math proof, debugging trace, logical argument) and asks which steps are most important or where the reasoning breaks down
- When reviewing chain-of-thought outputs from LLMs to identify reasoning quality beyond just answer correctness
- When the user wants to improve a step-by-step solution by pruning redundant steps or strengthening weak ones
- When debugging why a multi-step pipeline (data transformation, algorithm design, proof attempt) produces incorrect results despite some steps being valid
- When comparing two different solution paths to the same problem and deciding which reasoning chain is higher quality
- When building or refining prompts that require structured multi-step reasoning and the user wants to know which intermediate instructions actually contribute

## Key Technique

### Marginal Information Gain with Historical Watermark

Traditional evaluation gives binary feedback: the final answer is right or wrong. This discards all signal about *which intermediate steps* drove correctness or failure. Position-based heuristics (weighting later steps higher) also fail — empirically they correlate poorly (Spearman rho=0.254) with true step value because pivotal breakthroughs often happen early.

The MIG framework instead computes a **step-conditioned likelihood** for each step: how much does the reasoning prefix *through this step* increase confidence in the correct answer? Crucially, each step is compared against a **Monotonic Historical Watermark** — the highest confidence achieved by any prior step. Only steps that *exceed* this watermark receive positive credit. This means:

- A step that restates known information scores zero (no new progress)
- A step that makes a key insight scores high regardless of its position
- A step that introduces error scores negatively (confidence drops below watermark)
- The watermark only rises, so oscillating or circular reasoning is penalized

### Decoupled Process-Outcome Evaluation

The framework separates two concerns: (1) **process quality** — are the reasoning steps individually sound and progressive? and (2) **outcome correctness** — does the final answer match ground truth? These are evaluated with independent masks. This decoupling prevents a correct final answer from masking flawed intermediate reasoning, and prevents a wrong final answer from discrediting individually valid reasoning steps. A third gate activates only when *both* structural validity and answer correctness hold, identifying gold-standard reasoning chains suitable for direct reuse.

## Step-by-Step Workflow

1. **Parse the reasoning chain into discrete steps.** Segment the solution at natural boundaries: sentence breaks, numbered items, paragraph breaks, or explicit markers like `<step>` tags. Each segment becomes an independently evaluable unit.

2. **Identify the target outcome.** Determine the correct final answer or desired conclusion. If the user provides it, use it directly. If not, solve the problem independently first to establish ground truth.

3. **Compute step-conditioned confidence for each step.** For each step k, evaluate: "Given steps 1 through k, how likely is the correct answer?" Rate this on a 0-1 scale based on how much of the necessary information, logic, and computation has been established so far.

4. **Maintain the Historical Watermark.** Track the running maximum of step-conditioned confidence. Initialize at 0. After evaluating each step, update: `watermark_k = max(watermark_{k-1}, confidence_k)`.

5. **Compute Marginal Information Gain per step.** For each step: `MIG_k = max(0, confidence_k - watermark_{k-1})`. Steps that exceed the prior watermark get positive credit. Steps that don't get zero. Steps where confidence *drops* are flagged as potentially harmful.

6. **Flag regression points.** Any step where `confidence_k < watermark_{k-1}` indicates a reasoning regression — the step introduced error, confusion, or irrelevant content. Mark these explicitly for the user.

7. **Apply the dual-gate assessment.** Check two independent conditions: (a) Is the reasoning chain structurally valid (well-formed steps, logical connectives, no gaps)? (b) Is the final answer correct? Report the conjunction — chains passing both gates are high-quality exemplars; chains failing one reveal whether the problem is process or outcome.

8. **Produce the annotated credit report.** For each step, output: the step text, its MIG score, whether it's a breakthrough/neutral/regression, and a brief explanation of why.

9. **Recommend improvements.** Based on the credit distribution, suggest: removing zero-credit steps (redundant), strengthening regression steps (erroneous), and potentially reordering for clarity (though the method is position-agnostic for scoring).

10. **Verify with equivalence awareness.** If the problem admits multiple valid answer forms (e.g., "1.6" vs "8/5" vs "1 3/5"), ensure confidence scoring accepts all semantically equivalent forms to avoid false negatives.

## Concrete Examples

**Example 1: Debugging a Math Derivation**

User: "Score each step of my solution to: Solve 3x + 7 = 22"

```
Step 1: "We need to solve for x in 3x + 7 = 22"
Step 2: "First, note that 3x + 7 is a linear equation"
Step 3: "Subtract 7 from both sides: 3x = 15"
Step 4: "Recall that division is the inverse of multiplication"
Step 5: "Divide both sides by 3: x = 5"
```

Approach:
1. Target outcome: x = 5
2. Score each step's contribution to reaching x = 5

Credit Report:
```
Step 1: MIG = 0.05 | Neutral — Restates the problem, minimal new progress
         Watermark: 0.05
Step 2: MIG = 0.00 | Redundant — Classifies equation type, does not advance solution
         Watermark: 0.05
Step 3: MIG = 0.55 | BREAKTHROUGH — Key algebraic move, isolates 3x
         Watermark: 0.60
Step 4: MIG = 0.00 | Redundant — States a general fact without applying it
         Watermark: 0.60
Step 5: MIG = 0.40 | Breakthrough — Completes the solution, reaches x = 5
         Watermark: 1.00

Dual-gate: Structure PASS | Outcome PASS (x = 5 is correct)
Summary: Steps 3 and 5 carry 95% of the credit. Steps 2 and 4 can be
removed without loss. The reasoning is correct but inefficient.
```

**Example 2: Finding Where Code Debugging Logic Derails**

User: "I'm debugging why my binary search returns -1 for target=7 in [1,3,5,7,9]. Walk through my reasoning and find the flaw."

```
Step 1: "Initialize low=0, high=4"
Step 2: "mid = (0+4)//2 = 2, arr[2]=5, 5 < 7, so search right: low=3"
Step 3: "mid = (3+4)//2 = 3, arr[3]=7, found! Return 3"
Step 4: "Wait, my code returns -1, so maybe the array is 0-indexed wrong"
Step 5: "Let me assume 1-indexed: arr[1]=1, arr[2]=3, arr[3]=5, arr[4]=7, arr[5]=9"
Step 6: "With 1-indexing: mid=2, arr[2]=3, 3<7, low=3, mid=3, arr[3]=5, 5<7, low=4, mid=4, arr[4]=7, found"
Step 7: "So indexing isn't the problem. The bug must be in my comparison operator."
```

Credit Report:
```
Step 1: MIG = 0.10 | Neutral — Correct setup
         Watermark: 0.10
Step 2: MIG = 0.20 | Progress — Correct trace of first iteration
         Watermark: 0.30
Step 3: MIG = 0.25 | Progress — Correctly shows algorithm should find target
         Watermark: 0.55
Step 4: MIG = 0.00 | REGRESSION flag — Introduces a false hypothesis (indexing
         error) when step 3 already proved the algorithm logic works.
         Confidence drops to 0.35.
         Watermark: 0.55 (unchanged)
Step 5: MIG = 0.00 | Redundant — Pursues the false hypothesis from step 4
         Watermark: 0.55
Step 6: MIG = 0.00 | Redundant — Confirms 1-indexing also works, which was
         already evident
         Watermark: 0.55
Step 7: MIG = 0.15 | Partial recovery — Correctly discards the false hypothesis
         but jumps to another guess without evidence
         Watermark: 0.70

Dual-gate: Structure PASS | Outcome FAIL (root cause not identified)
Recommendation: After step 3 established the algorithm is correct in
theory, the user should examine their actual code (comparison operator,
loop bounds, return statement) rather than hypothesizing about indexing.
Steps 4-6 are a dead-end detour.
```

**Example 3: Comparing Two Solution Paths**

User: "Which proof that sqrt(2) is irrational is more efficient — proof A or proof B?"

Approach:
1. Parse each proof into steps
2. Compute MIG for each step in both proofs against the same target (establishing irrationality)
3. Compare total number of breakthrough steps vs redundant steps
4. Report which proof has higher credit density (sum of MIG / number of steps)

Output summary:
```
Proof A: 6 steps, 4 breakthroughs, 0 regressions, credit density = 0.78
Proof B: 9 steps, 4 breakthroughs, 1 regression, credit density = 0.49

Proof A is more efficient — same number of key insights delivered in
fewer steps with no regressions. Proof B's steps 5 and 7 are redundant
restatements, and step 8 introduces an unnecessary case split that is
immediately resolved.
```

## Best Practices

- **Do:** Score confidence based on semantic progress toward the answer, not on surface features like step length or mathematical notation density. A short step can be a major breakthrough.
- **Do:** Reset your watermark to 0 for each new reasoning chain. The watermark is per-chain, not global.
- **Do:** Accept equivalent answer forms when computing confidence. "0.5", "1/2", and "50%" should all register as progress toward any of these forms.
- **Do:** Separate your judgment of step quality from outcome correctness. A wrong final answer can still contain valuable intermediate reasoning.
- **Avoid:** Giving position-based credit (e.g., "later steps matter more"). The whole point of MIG is that a breakthrough at step 2 is as valuable as one at step 8.
- **Avoid:** Assigning negative MIG scores. The framework uses `max(0, ...)` — regressions are flagged separately but scored as zero credit, not negative credit.

## Error Handling

- **Ambiguous step boundaries:** If the reasoning chain is a single unpunctuated paragraph, segment at logical transition points (topic shifts, new operations, new sub-goals). State your segmentation explicitly so the user can adjust.
- **Unknown ground truth:** If you cannot independently verify the correct answer, perform the credit analysis conditionally: "Assuming the target answer is X, here is the credit distribution." Flag this assumption prominently.
- **Multiple valid solution paths:** When the problem has genuinely different valid approaches (e.g., induction vs. contradiction), a step that advances one path may seem irrelevant from another. Score against the path the reasoning chain is actually pursuing, not an alternative path.
- **Circular reasoning:** The watermark mechanism naturally handles this — if reasoning loops back, confidence won't exceed the watermark, so all circular steps score zero MIG. Flag the loop explicitly for the user.

## Limitations

- This technique evaluates reasoning *traces*, not the reasoning *process* itself. It cannot distinguish genuine insight from pattern matching if both produce the same step.
- The confidence scoring is a qualitative proxy, not a formal probability. Two evaluators may assign slightly different confidence values. The relative ordering (which steps are breakthroughs vs. redundant) is more reliable than exact scores.
- For very long chains (50+ steps), presenting per-step scores becomes unwieldy. In such cases, summarize by sections and only detail the top breakthroughs and worst regressions.
- The method assumes a single target outcome. For open-ended tasks (creative writing, brainstorming), where there is no single correct answer, MIG-based credit assignment is not applicable.
- This skill applies the *evaluation* framework from the paper, not the RL training loop. It does not fine-tune models — it helps humans and Claude assess and improve reasoning chains.

## Reference

Wang, X., Wang, W., Chen, K., Nimalsiri, N., & Halgamuge, S. (2026). *Discovering Process-Outcome Credit in Multi-Step LLM Reasoning.* arXiv:2602.01034v1. [https://arxiv.org/abs/2602.01034v1](https://arxiv.org/abs/2602.01034v1)

Key insight to look for: The Monotonic Historical Watermark mechanism and the empirical finding that content-aware step valuation (rho=0.623) dramatically outperforms position-based heuristics (rho=0.254) for identifying reasoning breakthroughs.