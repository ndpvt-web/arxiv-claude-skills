---
name: "rethinker-scientific-reasoning-rethinking"
description: "Solve hard scientific and technical reasoning problems using the ReThinker Solver-Critic-Selector loop with confidence-gated rethinking. Use when the user says 'reason through this carefully', 'solve this hard problem', 'think step by step and verify', 'scientific reasoning', 'double-check your answer', or 'I need a confident answer to this technical question'."
---

# ReThinker: Confidence-Aware Solver-Critic-Selector Reasoning

This skill enables Claude to tackle expert-level scientific, mathematical, and technical reasoning problems using the ReThinker framework from arXiv:2602.04496. Instead of producing a single answer in one pass, Claude decomposes reasoning into a **Solver-Critic-Selector** loop: first generating candidate solutions with retrieval-augmented reasoning, then applying multi-dimensional guided reflection to critique each solution, and finally using confidence-weighted selection to pick the best answer or trigger another rethinking cycle. This approach dynamically allocates effort — easy sub-problems resolve quickly while hard ones receive iterative refinement — producing more reliable answers on problems that demand expert-level analysis.

## When to Use

- When the user poses a hard scientific, mathematical, or engineering problem that requires multi-step reasoning (e.g., physics derivations, chemistry mechanisms, bio-informatics queries)
- When the user asks Claude to "verify" or "double-check" a technical answer and expects rigorous self-critique
- When solving problems that require combining information retrieval (searching docs, APIs, databases) with analytical reasoning
- When the user presents an ambiguous or multi-part technical question where a single-pass answer is likely to contain errors
- When debugging complex code logic where the root cause is non-obvious and multiple hypotheses must be evaluated
- When the user explicitly asks for confidence-aware reasoning or wants Claude to flag uncertainty in its answer

## Key Technique

**The core insight of ReThinker** is that not all reasoning steps deserve equal compute. Traditional chain-of-thought or multi-agent pipelines apply the same rigid process to every problem, wasting effort on easy parts and under-investing in hard parts. ReThinker replaces this with a **confidence-gated loop** across three roles:

1. **Solver**: Generates one or more candidate reasoning trajectories for a problem. Each trajectory may invoke tools (web search, code execution, calculation) as needed. The Solver does not commit to a single approach — it explores.
2. **Critic**: Evaluates each Solver trajectory along multiple dimensions: logical validity, factual accuracy, completeness, and consistency. The Critic produces structured feedback identifying specific weaknesses (e.g., "Step 3 assumes ideal gas behavior but the problem specifies high pressure") and suggests concrete corrections.
3. **Selector**: Assigns a confidence score to each critiqued trajectory and decides: (a) accept the best trajectory if confidence exceeds a threshold, (b) send the problem back to the Solver with the Critic's feedback for a guided rethinking cycle, or (c) decompose the problem into sub-problems if confidence remains low after multiple cycles.

**What makes this different from simple "self-reflection"** is the stage-wise separation of concerns and the confidence control mechanism. The Solver is never asked to judge its own work — that is the Critic's job. The Selector never generates solutions — it only routes. This separation prevents the common failure mode where a model "reflects" but rubber-stamps its original answer. The confidence threshold acts as a circuit breaker: it prevents both premature commitment (accepting a shaky answer) and infinite loops (rethinking forever).

## Step-by-Step Workflow

1. **Parse and classify the problem.** Read the user's question carefully. Identify the domain (math, physics, chemistry, CS theory, engineering, biology, etc.), the type of answer expected (numerical, proof, explanation, code), and any constraints or data provided. If the problem has multiple sub-questions, list them explicitly.

2. **Retrieve relevant context (Solver: retrieval phase).** Before reasoning, gather any external information needed. This may include: searching documentation, reading referenced files, recalling domain formulas or constants, or executing exploratory code to understand data. Record what you retrieved and why.

3. **Generate a candidate solution (Solver: reasoning phase).** Produce a complete step-by-step solution to the problem. Show all work — intermediate calculations, logical inferences, assumptions made. If the problem admits multiple approaches (e.g., energy methods vs. force methods in physics), generate at least two distinct solution trajectories.

4. **Critique each trajectory along multiple dimensions (Critic phase).** For each candidate solution, evaluate independently:
   - **Logical validity**: Does each step follow from the previous one? Are there gaps or non-sequiturs?
   - **Factual accuracy**: Are formulas, constants, definitions, and domain facts correct?
   - **Completeness**: Does the solution address all parts of the question? Are edge cases handled?
   - **Consistency**: Do multiple solution paths agree? If not, identify the point of divergence.
   Write explicit critique notes for each dimension. Do not simply say "looks correct" — identify at least one potential weakness or assumption to stress-test.

5. **Assign confidence and decide next action (Selector phase).** Rate your confidence in each trajectory on a scale:
   - **High (>0.85)**: The solution is logically sound, factually verified, and complete. Proceed to output.
   - **Medium (0.5–0.85)**: The solution is plausible but has identified weaknesses. Trigger a **guided rethinking cycle**: feed the Critic's specific feedback back into a new Solver pass that targets only the weak points.
   - **Low (<0.5)**: The approach is fundamentally flawed or the problem is under-specified. Decompose into sub-problems or ask the user for clarification.

6. **Execute guided rethinking (if triggered).** Using the Critic's concrete feedback, re-derive only the problematic portions of the solution. Do not restart from scratch — preserve the valid parts and surgically fix the identified issues. After revision, run the Critic phase again on the revised portions.

7. **Cap rethinking cycles.** Allow a maximum of 3 rethinking cycles. If confidence remains below 0.5 after 3 cycles, present the best available answer with an explicit uncertainty disclosure, explaining which aspects remain unresolved and why.

8. **Synthesize the final answer.** Present the selected solution cleanly, with:
   - The final answer prominently stated
   - Key reasoning steps preserved (not hidden)
   - A confidence qualifier (high / medium-with-caveats / low-with-disclosure)
   - Any assumptions that materially affect the answer

9. **Provide verification hooks.** Where possible, include a way for the user to independently verify the answer: a sanity check, a limiting-case analysis, a unit-dimension check, or executable code that reproduces the result.

## Concrete Examples

**Example 1: Multi-step physics problem**

User: "A 2 kg block sits on a 30-degree incline with coefficient of kinetic friction 0.3. It's connected via a massless string over a frictionless pulley to a 5 kg hanging block. Find the acceleration of the system and the tension in the string."

Approach:
```
SOLVER PASS 1:
- Draw free-body diagrams for both blocks.
- Block on incline: forces are gravity component along incline (mg sin30),
  normal force (mg cos30), friction (mu_k * N), and tension T.
- Hanging block: weight (Mg) and tension T.
- Write Newton's 2nd law for each block:
  Hanging:  Mg - T = Ma       =>  5(9.8) - T = 5a
  Incline:  T - mg sin30 - mu_k(mg cos30) = ma
            T - 2(9.8)(0.5) - 0.3(2)(9.8)(0.866) = 2a
            T - 9.8 - 5.09 = 2a  =>  T - 14.89 = 2a
- Add equations: 49 - 14.89 = 7a  =>  a = 4.87 m/s^2
- T = 49 - 5(4.87) = 24.65 N

CRITIC:
- Logical validity: Equations are correctly set up assuming the hanging
  block accelerates downward and the incline block accelerates upward.
  Need to verify friction direction — if block moves up the incline,
  friction acts downward along incline. Correct as written.
- Factual accuracy: cos(30) = 0.866, sin(30) = 0.5. Check: 0.3 * 2 *
  9.8 * 0.866 = 5.09. Confirmed.
- Completeness: Both unknowns (a and T) are found. Units are consistent.
- Consistency: Sanity check — a < g, T < Mg, T > 0. All satisfied.

SELECTOR: Confidence = 0.92 (High). Accept.
```

Output:
```
Acceleration: 4.87 m/s^2 (hanging block down, incline block up the slope)
Tension: 24.65 N
Confidence: High

Verification: The acceleration is less than g (9.8), which makes sense
since the incline block partially counterbalances the hanging block.
Tension is between 0 and Mg (49 N), consistent with the system accelerating
but not in free fall.
```

**Example 2: Debugging a subtle algorithmic bug**

User: "My binary search returns -1 for values I know are in the array. Here's my code:"
```python
def binary_search(arr, target):
    lo, hi = 0, len(arr)
    while lo < hi:
        mid = (lo + hi) // 2
        if arr[mid] == target:
            return mid
        elif arr[mid] < target:
            lo = mid
        else:
            hi = mid
    return -1
```

Approach:
```
SOLVER PASS 1 — Hypothesis A: Off-by-one in bounds.
- lo starts at 0, hi at len(arr). This is a half-open [lo, hi) convention.
- When arr[mid] < target, we set lo = mid. But mid was already checked,
  so lo should be mid + 1. Otherwise, when lo = mid (which happens when
  hi = lo + 1), the loop never terminates — it's an infinite loop, not
  a wrong answer.

Wait — the user says it returns -1, not that it hangs. Let me re-examine.

SOLVER PASS 2 — Hypothesis B: The infinite loop eventually terminates
  due to some other condition? No — lo < hi is the only loop guard.
  If lo = mid and hi = lo + 1, then lo < hi remains true forever.

CRITIC:
- The user reports getting -1, but analysis shows an infinite loop should
  occur instead. Possible explanations: (1) the user has a timeout or
  recursion limit that converts hangs into -1, (2) the bug manifests
  differently for their specific inputs, (3) the code shown is slightly
  different from what they actually run.
- The fundamental bug is clear: `lo = mid` should be `lo = mid + 1`.
  This is a classic binary search error.
- Completeness: Should also check the `hi = mid` branch — since we use
  half-open convention, hi = mid is correct (excludes mid).

SELECTOR: Confidence = 0.88 on the fix. Medium-high on the symptom
  explanation (the user may be experiencing an infinite loop misreported
  as -1). Accept fix, flag discrepancy.
```

Output:
```
Bug: On line `lo = mid`, when `arr[mid] < target`, `mid` has already been
checked so it should be excluded. Change to `lo = mid + 1`.

Fixed code:
    elif arr[mid] < target:
        lo = mid + 1

Note: With the original code, certain inputs will cause an infinite loop
rather than returning -1 (specifically when hi - lo == 1 and arr[lo] <
target). If you're seeing -1 returned, you may have a timeout wrapper, or
the bug manifests before the infinite-loop case for your test inputs.

Verification: Test with arr = [1, 3, 5, 7], target = 3. Original code
loops forever when lo=0, hi=1. Fixed code correctly returns index 1.
```

**Example 3: Chemistry equilibrium with retrieval**

User: "Calculate the pH of a 0.1 M solution of acetic acid (Ka = 1.8 x 10^-5)."

Approach:
```
SOLVER:
- Acetic acid is a weak acid: CH3COOH ⇌ CH3COO- + H+
- ICE table: initial [HA]=0.1, [A-]=0, [H+]=0
  Change: -x, +x, +x
  Equilibrium: 0.1-x, x, x
- Ka = x^2 / (0.1 - x) = 1.8e-5
- Assume x << 0.1: x^2 = 1.8e-6, x = 1.34e-3
- Check assumption: 1.34e-3 / 0.1 = 1.34% < 5%. Valid.
- pH = -log(1.34e-3) = 2.87

CRITIC:
- Logical validity: ICE table correctly set up. Quadratic approximation
  valid (1.34% < 5% threshold).
- Factual accuracy: Ka value matches user input. Log calculation:
  -log(1.34e-3) = 3 - log(1.34) = 3 - 0.127 = 2.87. Correct.
- Completeness: Single-step equilibrium fully addressed.
- Consistency: pH between 1 and 7 for weak acid — reasonable.

SELECTOR: Confidence = 0.95. Accept.
```

Output:
```
pH = 2.87

Method: Weak acid equilibrium with small-x approximation.
[H+] = sqrt(Ka * C) = sqrt(1.8e-5 * 0.1) = 1.34 x 10^-3 M
Approximation validity: x/C = 1.34% (< 5% threshold, so valid).
```

## Best Practices

- **Do** generate multiple solution trajectories for problems where the approach is non-obvious. Comparing two independent derivations is the single most effective way to catch errors.
- **Do** write explicit Critic notes even when the solution seems obviously correct. The act of checking forces attention to details that silent confidence skips over.
- **Do** use the confidence threshold honestly. If you are not genuinely confident, say so — a calibrated "I'm 60% sure" is more useful than a falsely assured wrong answer.
- **Do** preserve valid reasoning when rethinking. Surgical revision of weak steps is faster and less error-prone than restarting from scratch.
- **Avoid** rubber-stamp critiques like "the reasoning looks correct." Every Critic pass must identify at least one concrete aspect to stress-test, even if it ultimately holds up.
- **Avoid** exceeding 3 rethinking cycles. Diminishing returns set in quickly; after 3 cycles, present your best answer with honest uncertainty rather than spinning.
- **Avoid** conflating the Solver and Critic roles. Never critique your answer in the same breath as generating it. Finish the full solution first, then switch to adversarial evaluation.

## Error Handling

| Failure Mode | Detection | Recovery |
|---|---|---|
| Solver produces contradictory trajectories | Critic finds divergent answers for the same quantity | Trace both paths to find the divergence point; re-derive that specific step with extra care |
| Confidence stays low after 3 cycles | Selector repeatedly scores < 0.5 | Present partial results, identify which sub-problems are unresolved, and ask the user for clarification or additional constraints |
| Retrieved information conflicts with domain knowledge | Critic flags factual inconsistency | Prefer established domain knowledge over retrieved content; flag the conflict to the user |
| Problem is under-specified | Solver must make unstated assumptions | Make each assumption explicit, solve conditionally, and present sensitivity analysis showing how the answer changes under alternative assumptions |
| Rethinking cycle fixes one issue but introduces another | Critic finds new errors in revised sections | Diff the revision against the original to isolate regression; revert the regressed portion and re-attempt only that fix |

## Limitations

- **Not suited for purely creative or subjective tasks.** The Solver-Critic-Selector loop is designed for problems with verifiable answers. For creative writing or opinion questions, the Critic has no objective ground truth to evaluate against.
- **Confidence scores are self-assessed.** Claude's confidence estimates are heuristic, not calibrated probabilities. They are useful for relative ranking (this trajectory vs. that one) but should not be interpreted as true statistical confidence intervals.
- **Retrieval quality is a bottleneck.** If the problem requires specialized knowledge not in Claude's training data and retrieval fails to surface it, additional rethinking cycles will not compensate for missing information.
- **Overhead is not free.** The multi-pass approach uses significantly more tokens than a single-pass answer. For straightforward questions where Claude's first answer is reliably correct, this framework adds cost without benefit. Reserve it for problems where single-pass accuracy is genuinely insufficient.
- **Maximum 3 cycles is a heuristic.** Some problems may need more iteration; others may need less. The cap prevents runaway loops but is not a principled optimality guarantee.

## Reference

**Paper**: [ReThinker: Scientific Reasoning by Rethinking with Guided Reflection and Confidence Control](https://arxiv.org/abs/2602.04496v1) — Tang et al., 2026. Key sections: the Solver-Critic-Selector architecture (Section 3), the confidence-gated rethinking loop (Section 3.2), and ablation results showing that removing any single component (Critic, confidence gating, or multi-trajectory generation) degrades accuracy on HLE and GAIA benchmarks.