---
name: "verge-formal-refinement-guidance"
description: "Iterative verification-guided reasoning that decomposes answers into atomic claims, classifies and routes them to formal (SMT/logic) or consensus-based verification, localizes errors via Minimal Correction Subsets, and refines until convergence. Use when: 'verify my reasoning step by step', 'check this logic for contradictions', 'formally verify this argument', 'find the flaw in this proof', 'validate these claims against constraints', 'refine this answer until it is logically consistent'."
---

# VERGE: Formal Refinement and Guidance Engine for Verifiable Reasoning

This skill enables Claude to apply the VERGE neurosymbolic verification framework to complex reasoning tasks. Instead of generating a single-pass answer and hoping it is correct, Claude decomposes its own output into individually verifiable atomic claims, classifies each claim by type (mathematical, logical, temporal, commonsense, vague, probabilistic), routes each to the appropriate verification strategy (formal logic checking or multi-perspective consensus), localizes errors to the exact conflicting claims using Minimal Correction Subsets, and iteratively refines the answer using structured feedback until it meets acceptance criteria or converges. This produces reasoning with explicit verification status for every claim, not just a confidence score.

## When to Use

- When the user asks Claude to solve a logic puzzle, constraint satisfaction problem, or scheduling problem with hard constraints that must all hold simultaneously
- When the user wants Claude to verify a legal, mathematical, or engineering argument for internal contradictions
- When the user asks Claude to debug a chain of reasoning and identify exactly which step is wrong
- When the user provides a set of facts or rules and asks whether a conclusion follows
- When the user needs an answer to a multi-step word problem and wants proof that intermediate steps are consistent
- When the user asks Claude to validate business rules, policy constraints, or data integrity conditions against a proposed action
- When the user asks Claude to iteratively improve an answer rather than give a one-shot response

## Key Technique

VERGE's core insight is that LLM outputs are not monolithic -- they are composed of independent atomic claims that can be individually verified and selectively revised. Rather than treating verification as a binary pass/fail on the entire answer, VERGE decomposes the answer into minimal, irreducible claims (e.g., "Alice is older than Bob", "The meeting is on Tuesday", "Revenue increased 12%"), then checks each one against the known constraints and against each other. This converts opaque reasoning into a transparent, auditable structure.

The framework makes a pragmatic distinction between claims that can be formally verified and those that cannot. Mathematical, logical, and temporal claims are autoformalized into first-order logic (specifically SMT-LIB2 fragments like QF_LIA for integer arithmetic) and checked by an SMT solver (Z3) for entailment, contradiction, or mere possibility. Commonsense, vague, and probabilistic claims -- which resist formalization -- are instead verified through multi-perspective consensus (querying the claim from multiple angles and checking agreement). This semantic routing avoids the trap of trying to formalize everything, which either fails or produces meaningless formalizations.

When contradictions are found, VERGE does not simply report "your answer is wrong." It computes the Minimal Correction Subset (MCS): the smallest set of claims whose removal restores logical consistency. This tells the refinement step exactly which claims to revise, not the entire answer. The system then feeds structured feedback (which claims conflict, what the solver found, which formalizations were low-confidence) back into a new generation pass. This loop repeats until the aggregate verification score exceeds a threshold (0.75) with joint satisfiability, or until score improvement falls below 0.01 (convergence), or a maximum iteration count is reached.

## Step-by-Step Workflow

1. **Decompose the answer into atomic claims.** Take the LLM's initial response and break it into a numbered list of minimal, independently evaluable statements. Each claim should be irreducible -- it asserts exactly one fact, relationship, or inference. Remove hedging language; extract the underlying assertion.

2. **Classify each claim by semantic type.** Label every claim as one of: Mathematical (quantitative relationships, arithmetic), Logical (if-then, all/some, negation), Temporal (ordering, duration, deadlines), Commonsense (world knowledge, typical behavior), Vague (subjective, qualitative), or Probabilistic (likelihood, frequency). This classification determines the verification route.

3. **Formalize hard claims into logic.** For claims classified as Mathematical, Logical, or Temporal: extract typed entities (e.g., `age: Int`, `day: Enum{Mon..Sun}`), translate the claim into a formal constraint (e.g., `age_alice > age_bob`), and translate the known context/premises into a constraint set. State the logic fragment being used (propositional, integer arithmetic, uninterpreted functions). Perform a round-trip check: restate the formula in natural language and confirm it matches the original claim.

4. **Verify hard claims via solver reasoning.** For each formalized claim, check three conditions against the context constraints: (a) Entailed -- the claim necessarily follows (negation is unsatisfiable under context), (b) Contradictory -- the claim is impossible (conjunction with context is unsatisfiable), (c) Possible -- consistent but not forced. Assign status accordingly.

5. **Verify soft claims via multi-perspective consensus.** For Commonsense, Vague, and Probabilistic claims: evaluate the claim from at least 3 independent angles (e.g., rephrase the question, consider counterexamples, check against known heuristics). Mark as Supported if perspectives agree, Contradictory if they conflict, or Possible if inconclusive.

6. **Check joint consistency.** Beyond individual claim verification, check whether all non-contradictory claims are mutually consistent when combined. This catches cases where each claim is individually plausible but they cannot all be true simultaneously (e.g., scheduling conflicts, over-constrained systems).

7. **Compute the aggregate verification score.** Assign per-claim scores: Entailed = 1.0, Supported = 0.9, Possible = 0.7, Contradictory = 0.0. Compute the mean score, then apply a variance penalty: `Score = mean * max(0.5, 1.0 - stddev / (mean + 0.01))`. This penalizes answers with a mix of strong and weak claims.

8. **Localize errors via Minimal Correction Subsets.** If contradictions exist or joint consistency fails, identify the smallest set of claims whose removal restores consistency. Use a greedy approach: sort claims by verification confidence (highest first), then iteratively add claims to the "keep" set, checking consistency at each step. Claims that break consistency when added are flagged for revision.

9. **Generate structured feedback and refine.** Produce explicit feedback listing: (a) which specific claims are contradictory and why, (b) which claims form the MCS (the revision targets), (c) which formalizations had low confidence. Feed this back into a new answer generation pass that preserves verified claims and revises only the flagged ones.

10. **Iterate until acceptance or convergence.** Repeat steps 1-9. Stop when: the aggregate score >= 0.75 AND joint consistency holds, OR the score improvement between iterations < 0.01, OR a maximum of 3 iterations is reached. Return the best-scoring answer with its full claim-level verification report.

## Concrete Examples

**Example 1: Logic Puzzle Verification**

User: "There are 5 houses in a row. Each has a different color and a different pet. The red house is immediately to the left of the green house. The cat owner lives in the blue house. The dog owner lives next to the blue house. The fish owner lives in the red house. Which house has the bird?"

Approach:
1. Generate initial answer: "The bird is in house 3 (the yellow house)."
2. Decompose into atomic claims:
   - C1: The red house is at position X (X < 5, since green is to its right)
   - C2: The green house is at position X+1
   - C3: The blue house contains a cat
   - C4: The dog owner is adjacent to the blue house
   - C5: The fish owner is in the red house
   - C6: The bird is in house 3
   - C7: House 3 is yellow
3. Classify: all Logical (constraint satisfaction). Formalize as integer constraints with distinctness.
4. Verify each against the constraint set. Suppose C6 contradicts the joint constraint set.
5. MCS identifies C6 ("bird is in house 3") as the minimal correction target.
6. Refine: regenerate with feedback "C6 is contradictory under the full constraint set; revise the bird's position."
7. New answer: "The bird is in house 5 (the white house)." Re-verify: all claims Entailed. Score = 1.0. Accept.

Output:
```
Verified Answer: The bird is in house 5 (the white house).

Claim Verification Report:
  C1: Red house at position 2        [Entailed]
  C2: Green house at position 3      [Entailed]
  C3: Blue house has cat             [Entailed]
  C4: Dog adjacent to blue house     [Entailed]
  C5: Fish in red house              [Entailed]
  C6: Bird in house 5                [Entailed]
  C7: House 5 is white               [Entailed]

Aggregate Score: 1.0 (joint consistency: PASS)
Iterations: 2
```

**Example 2: Business Rule Validation**

User: "Our policy says: premium customers get free shipping on orders over $50, new customers get 10% off their first order, and no order can combine free shipping with a percentage discount. A new premium customer places a $75 first order. What discount do they get?"

Approach:
1. Generate initial answer: "They get both free shipping and 10% off."
2. Decompose:
   - C1: Customer qualifies as premium (given)
   - C2: Order exceeds $50 (given: $75)
   - C3: Customer qualifies as new (given: first order)
   - C4: Premium + over $50 -> free shipping
   - C5: New customer -> 10% off first order
   - C6: Customer receives free shipping AND 10% off
3. Classify: C1-C3 are factual (Entailed from context). C4-C5 are Logical (policy rules). C6 is Logical.
4. Formalize the no-combination rule: `NOT (free_shipping AND pct_discount)`. C6 asserts both. Contradiction detected.
5. MCS: C6 is the minimal correction target. C4 and C5 individually hold, but C6 (their conjunction) violates the combination rule.
6. Feedback: "C6 contradicts the no-combination policy. The customer qualifies for both individually but must choose one."
7. Refined answer acknowledges the conflict and presents the choice.

Output:
```
Verified Answer: The customer qualifies for both free shipping
(as a premium customer with a $75 order) and 10% off (as a new
customer), but policy prohibits combining them. They must choose
one: free shipping (saves ~$8 shipping) or 10% off ($7.50 discount).

Claim Verification Report:
  C1: Customer is premium             [Entailed]
  C2: Order is $75 (> $50)            [Entailed]
  C3: Customer is new                 [Entailed]
  C4: Qualifies for free shipping     [Entailed]
  C5: Qualifies for 10% off           [Entailed]
  C6: Cannot combine both discounts   [Entailed]
  C7: Customer must choose one        [Entailed]

Aggregate Score: 1.0 (joint consistency: PASS)
Iterations: 2
```

**Example 3: Debugging a Flawed Proof**

User: "I wrote this proof that all triangles are equilateral. Find the flaw: (1) Take any triangle ABC. (2) Let the angle bisector of A meet BC at D. (3) Let the perpendicular bisector of BC meet AD at P. (4) Drop perpendiculars from P to AB and AC, meeting at E and F. (5) Then PE = PF (P is on angle bisector). (6) And PB = PC (P is on perpendicular bisector of BC). (7) So triangles PEB and PFC are congruent (RHS). (8) Therefore EB = FC. (9) Also AE = AF (P on angle bisector, PE = PF). (10) So AB = AE + EB = AF + FC = AC. (11) Similarly all sides are equal."

Approach:
1. Decompose into claims C1-C11 as stated.
2. Classify: C1-C4 are constructions (Logical/Geometric). C5-C10 are Logical (geometric deductions). C11 is the conclusion.
3. Verify each deduction:
   - C5: PE = PF because P lies on the angle bisector of A. [Entailed -- angle bisector equidistance theorem]
   - C6: PB = PC because P lies on perpendicular bisector of BC. [Entailed]
   - C7: Triangles PEB, PFC congruent by RHS. [Possible -- requires E and F to be on the segments AB and AC, not their extensions]
   - C9: AE = AF. [Entailed if PE = PF and P is on angle bisector]
   - C10: AB = AE + EB. [Possible -- only if E is between A and B]
4. MCS identifies C10 (and implicitly C8): the assumption that E lies between A and B (and F between A and C) is not guaranteed. For non-equilateral triangles, P falls outside the triangle and E, F lie on extensions of the sides, making AB = AE - EB, not AE + EB.
5. Structured feedback: "The flaw is in step 10. The point P can lie outside triangle ABC for non-equilateral triangles, causing E to fall on the extension of AB beyond B. In that case AB = AE - EB, not AE + EB, breaking the chain of equalities."

Output:
```
Error Localization:
  Step 10 (C10): AB = AE + EB         [Contradictory]
  Reason: Assumes E lies between A and B on segment AB.
  For non-equilateral triangles, the intersection point P
  falls outside the triangle, placing E on the extension
  of AB beyond B. The correct relation becomes AB = AE - EB.

All other steps are individually valid. The fallacy is a
hidden geometric assumption about point position, not a
logical inference error.
```

## Best Practices

- **Do:** Always decompose before verifying. Verifying a monolithic answer gives you only a binary signal. Decomposition gives you per-claim diagnostics.
- **Do:** Perform the round-trip check when formalizing. Restate the formal constraint in natural language and confirm it matches the original claim. Mistranslation between natural language and logic is the most common failure mode.
- **Do:** Check joint consistency, not just individual claims. Five individually plausible scheduling claims can be mutually impossible.
- **Do:** Preserve verified claims across refinement iterations. Only revise claims flagged by MCS. Rewriting the entire answer risks introducing new errors in previously correct sections.
- **Avoid:** Attempting to formalize commonsense or subjective claims. "The weather is nice" cannot be meaningfully rendered in FOL. Route these to consensus verification instead.
- **Avoid:** Treating the aggregate score as a confidence percentage. A score of 0.85 means "most claims are verified with moderate variance," not "85% likely correct." The variance penalty matters -- an answer with scores [1.0, 1.0, 0.0] is worse than [0.7, 0.7, 0.7].

## Error Handling

- **Formalization failure:** If a claim resists clean formalization (ambiguous quantifiers, missing type information), flag it as low-confidence and fall back to soft verification. Do not force a bad formalization -- a wrong formula verified as "Entailed" is worse than an unformalized claim verified by consensus.
- **Solver timeout / undecidable fragment:** If the constraint set is too complex for tractable solving (e.g., nonlinear arithmetic, quantifier alternation), drop to the soft verification path and note the limitation in the report.
- **Convergence without acceptance:** If the score plateaus below the 0.75 threshold, report the best answer found with its full claim-level breakdown. Highlight the specific unresolved claims so the user can apply domain judgment.
- **Decomposition ambiguity:** If the answer contains compound claims that resist clean splitting (e.g., "A because B implies C"), err on the side of finer decomposition. It is better to have redundant atomic claims than to miss a hidden contradiction inside a compound statement.
- **Circular dependencies between claims:** If claim C3 depends on C5 which depends on C3, flag the cycle explicitly. Circular reasoning cannot be verified by checking claims individually -- it requires recognizing the dependency structure.

## Limitations

- This technique adds significant overhead to every response. Use it for high-stakes reasoning where correctness matters more than speed, not for casual conversation or creative writing.
- Autoformalization is imperfect. Complex natural language claims involving metaphor, irony, implicit context, or domain-specific jargon may be mis-formalized. Always review formalizations critically.
- The approach is strongest on closed-world problems where all relevant facts are stated. Open-world reasoning (where missing information matters) is harder to verify because the solver can only check against what it knows.
- Commonsense claims verified by consensus lack formal guarantees. A unanimously agreed-upon commonsense claim can still be wrong if the underlying model has a systematic bias.
- Maximum iteration count must be bounded. Without a cap, adversarial or genuinely unsatisfiable constraint sets could cause infinite refinement loops. Three iterations covers most convergent cases.

## Reference

**Paper:** [VERGE: Formal Refinement and Guidance Engine for Verifiable LLM Reasoning](https://arxiv.org/abs/2601.20055v1) (Singh et al., 2026). See Section 3 for the full verification pipeline, Section 3.4 for MCS-based error localization, and Appendix A for the iterative refinement algorithm pseudocode.