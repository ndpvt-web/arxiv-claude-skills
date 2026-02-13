---
name: "towards-autonomous-mathematics-research"
description: "Iterative generate-verify-revise agent for mathematical research problems. Implements the Aletheia loop: decompose a hard math problem, generate candidate proofs, verify with separate critical passes, revise on failure, and integrate literature search. Use when: 'solve this open math problem', 'prove this conjecture', 'verify this proof attempt', 'search for counterexamples', 'tackle this research-level math question', 'help me with this PhD-level exercise'."
---

# Towards Autonomous Mathematics Research (Aletheia Pattern)

This skill enables Claude to act as a mathematical research agent following the Aletheia architecture from Google DeepMind. The core idea: decouple generation from verification, iterate with revision, and use tools aggressively to ground claims in real literature. Rather than attempting a proof in a single pass, the agent cycles through generate-verify-revise loops, treating its own outputs as drafts to be critically examined. This pattern extends naturally from competition math to research-level problems by adding literature navigation, computational experimentation, and honest failure admission.

## When to Use

- When the user asks to prove a theorem, lemma, or conjecture that requires multi-step reasoning
- When the user wants to verify or find flaws in an existing proof attempt
- When the user asks to explore open problems from a database (e.g., Erdos conjectures, OEIS sequences, open problem lists)
- When the user needs to search mathematical literature to contextualize a result or find prior art
- When the user asks for computational experiments to test a conjecture before attempting proof
- When the user wants to decompose a hard research question into tractable sub-problems
- When the user asks to generate and rigorously check mathematical content for a paper

## Key Technique: The Generate-Verify-Revise Loop

Aletheia's central insight is that **separating generation from verification dramatically improves mathematical reasoning**. A single-pass approach conflates creative exploration with critical evaluation. Instead, the agent first generates a candidate solution with extended reasoning, then switches to a distinct verification mode that scrutinizes the output for logical gaps, unstated assumptions, and incorrect citations. When verification finds flaws, a revision pass incorporates the specific critique to produce an improved attempt. This cycle repeats until the solution passes verification or the agent honestly admits failure.

The second key insight is **inference-time scaling**: allocating more compute during problem-solving (not training) yields better results, following a predictable scaling law. For practical purposes, this means the agent should spend more reasoning effort on harder problems rather than rushing to an answer. On research-grade problems, scaling alone is insufficient -- tool use (literature search, computation, counterexample search) becomes essential to bridge the gap.

The third insight is **honest failure admission**. Aletheia's verification stage can reject a solution entirely, causing the agent to report that it cannot solve the problem rather than presenting a flawed proof confidently. In their evaluation of 700 Erdos open problems, 68.5% of candidate solutions were fundamentally flawed -- the system's ability to catch and discard these was more valuable than its ability to generate correct ones.

## Step-by-Step Workflow

1. **Parse and formalize the problem statement.** Extract the precise mathematical claim, identify all quantifiers, define all terms, and restate the problem in unambiguous notation. Flag any ambiguity in the original statement -- specification gaming (choosing the easiest interpretation) is a documented failure mode.

2. **Search for existing results.** Before attempting a proof, search for prior work. Use web search to find whether the problem (or a stronger/weaker version) has been solved. Check relevant databases (arXiv, MathSciNet, OEIS, Erdos Problems, MathOverflow). Record what is known and what tools/techniques have been applied.

3. **Classify the problem's difficulty and select a strategy.** Categorize as: (a) exercise-level (direct application of known techniques), (b) competition-level (requires clever but elementary arguments), or (c) research-level (requires novel ideas or extensive background). For (a), proceed directly. For (b), attempt elementary self-contained arguments first. For (c), decompose into sub-problems and identify which require literature vs. computation vs. new ideas.

4. **Generate a candidate solution (Generation Pass).** Produce a complete proof attempt with full reasoning shown. Use extended chain-of-thought. For research-level problems, explicitly state which known results you are invoking and where novel arguments begin. Do not suppress intermediate steps.

5. **Verify the candidate critically (Verification Pass).** Switch to adversarial mode. Read the candidate solution as a skeptical reviewer. For each logical step, ask: Does this follow? Is this assumption justified? Is this citation real and correctly applied? Are there edge cases? Write a structured verification report listing each issue found, categorized as: (a) fatal flaw, (b) gap requiring additional argument, (c) minor issue.

6. **Run computational checks where applicable.** Use Python/SageMath/SymPy to test the claim on small cases, search for counterexamples, verify combinatorial identities, or compute numerical evidence. Computational validation is especially valuable for conjectures involving inequalities, bounds, or counting arguments.

7. **Revise based on verification feedback (Revision Pass).** Address each issue from the verification report. For fatal flaws, attempt an alternative approach rather than patching. For gaps, fill them with explicit sub-proofs. For citation errors, find the correct reference or prove the claimed result from scratch. Produce a complete revised solution.

8. **Repeat the verify-revise cycle.** Run verification again on the revised solution. Continue until either: (a) verification passes with no fatal flaws or gaps, (b) you have exhausted 3-4 revision cycles without convergence, or (c) the problem is determined to be beyond current capability.

9. **Admit failure honestly when appropriate.** If the problem resists solution after multiple cycles, report this clearly. State what partial progress was made, which sub-problems were resolved, and where the proof breaks down. A honest "I cannot solve this" with partial progress is far more valuable than a flawed proof presented with false confidence.

10. **Document the result with an interaction card.** Classify the result along two axes: Autonomy (H = primarily human-guided, C = genuine collaboration, A = essentially autonomous) and Significance (0 = exercise, 1 = minor technique, 2 = publication-grade, 3 = major advance). State clearly what the AI contributed vs. what required human input.

## Concrete Examples

**Example 1: Verifying a proof attempt**

```
User: I think I proved that every graph with minimum degree >= n/2 has a
Hamiltonian cycle. Can you check my proof? [attaches proof]

Approach:
1. Parse the claim -- this is Dirac's theorem (1952), a well-known result.
   Inform the user of the prior art.
2. Read the user's proof line by line in Verification Pass mode.
3. Check each step:
   - Does the base case hold?
   - Is the inductive step valid?
   - Are all edge cases (small n, equality in the bound) handled?
4. Run computational spot-checks: generate random graphs with min
   degree = n/2 for n = 6, 8, 10 and verify Hamiltonian cycles exist.
5. Produce a structured verification report.

Output:
## Verification Report

**Claim**: Every graph on n >= 3 vertices with δ(G) >= n/2 has a
Hamiltonian cycle.

**Status**: Known result (Dirac, 1952).

**Step-by-step review**:
- Step 1 (longest path argument): Correct.
- Step 2 (rotation-extension): Gap found -- you assume the endpoints
  of the longest path have disjoint neighborhoods, but this requires
  proof. Specifically, if u and v are endpoints of a longest path P,
  and N(u) ∩ V(P) and N(v) ∩ V(P) overlap in a specific way, the
  rotation produces a cycle but not necessarily a Hamiltonian one.
- Step 3 (pigeonhole closure): Correct given Step 2 is fixed.

**Computational check**: Tested 1000 random graphs, all confirmed.

**Recommendation**: Fix Step 2 by using the pigeonhole argument
directly: |N(u) ∩ V(P)| + |N(v) ∩ V(P)| >= n > |V(P)|, so there
exists i where v_i ∈ N(u) and v_{i-1} ∈ N(v).
```

**Example 2: Tackling an open combinatorial problem**

```
User: Erdos conjectured that every set of n points in the plane, no three
collinear, determines at least n/2 distinct distances. Can you
investigate this?

Approach:
1. Search literature: This is the Erdos distinct distances problem.
   Guth-Katz (2015) proved Ω(n/log n) distinct distances, nearly
   resolving it. The exact n/2 lower bound remains open.
2. Report the state of the art to the user before attempting anything.
3. Generation Pass: Attempt to improve the constant in known bounds
   using the Guth-Katz polynomial partitioning framework.
4. Verification Pass: Check whether the attempted improvement actually
   yields anything new or just rediscovers known results.
5. Computational check: For small n (5-15), enumerate point
   configurations minimizing distinct distances to build intuition.
6. Honest assessment of what is achievable.

Output:
## Investigation Report: Erdos Distinct Distances

**Problem**: n points in general position determine >= n/2 distances.
**Classification**: Research-level (Significance 3), beyond current
AI capability for full resolution.

**State of the art**: Guth-Katz (2015) proved Ω(n/log n), which is
tight up to the log factor. The conjecture for the exact constant
remains open.

**Computational exploration**: For n=5 through n=12, I verified the
conjecture holds. The extremal configurations are:
- n=5: regular pentagon (5 points, 2 distances -- but this has 3
  collinear? No, pentagon is fine. 2 < 5/2 = 2.5... actually the
  pentagon gives only 2 distinct distances.)

**Correction (Verification Pass)**: Wait -- the regular pentagon gives
exactly 2 distinct distances with 5 points, and 5/2 = 2.5, so 2 < 2.5
would be a counterexample. But the conjecture actually states "at least
floor(n/2)" or the problem has additional constraints. Let me re-check
the exact statement...

**Revised finding**: The precise conjecture is that the minimum number
of distinct distances is Ω(n/√log n). The "n/2" version applies to
convex position. This illustrates why precise problem parsing (Step 1)
is critical.

**Autonomy card**: Level H0 (human-guided, exercise-level literature
review). No novel mathematical contribution.
```

**Example 3: Generating a research-level computation**

```
User: I'm studying eigenweights for the group GL_3 over a finite field.
Can you compute the eigenweight for the representation associated to
the partition (2,1)?

Approach:
1. Search literature for eigenweights, GL_3, finite fields, and the
   partition (2,1). Identify the relevant algebraic combinatorics
   framework.
2. Formalize: eigenweights are structure constants in the Hecke algebra
   or related algebra. Identify the precise definition from the user's
   context or standard references.
3. Generation Pass: Compute the eigenweight using the combinatorial
   formula (e.g., involving Hall polynomials, Kostka numbers, or
   character values).
4. Verification Pass: Cross-check the computation against known special
   cases. For GL_2, eigenweights are well-understood -- verify the
   method reproduces those.
5. Computational check: Implement the computation in Python/SageMath
   and verify numerically for small field sizes (q = 2, 3, 5).
6. Present the result with full derivation.

Output:
## Eigenweight Computation: GL_3, partition (2,1)

**Setup**: Working over F_q. The eigenweight for the partition
λ = (2,1) of 3 is computed via [specific formula from literature].

**Computation**:
[Full step-by-step algebraic derivation]

**SageMath verification**:
```python
# Verify for q = 2, 3, 5
from sage.all import *
for q in [2, 3, 5]:
    result = compute_eigenweight(q, [2,1])
    expected = q^2 + q  # derived formula
    assert result == expected, f"Failed for q={q}"
    print(f"q={q}: eigenweight = {result} ✓")
```

**Result**: The eigenweight equals q^2 + q.

**Autonomy card**: Level A1 (autonomous computation, minor technique).
```

## Best Practices

**Do:**
- Always run a separate verification pass after generating a solution -- never trust the first output
- Search for existing results before attempting a proof; rediscovery wastes effort and risks presenting known work as novel
- Use computational checks as a complement to logical reasoning, especially for inequalities and combinatorial claims
- Admit failure explicitly when verification repeatedly finds fatal flaws; partial progress with honest assessment beats a flawed proof
- State precisely which known results you invoke and verify that citations are real (hallucinated citations are a documented failure mode)
- Require elementary, self-contained arguments when possible; avoid invoking heavy machinery unless truly necessary

**Avoid:**
- Presenting a proof that has not survived at least one full verification pass
- Choosing the easiest interpretation of an ambiguous problem statement (specification gaming)
- Citing theorems you cannot state precisely -- if you cannot write the exact statement, do not invoke it
- Running more than 3-4 generate-verify-revise cycles on the same approach; after 3 failures, try a fundamentally different strategy
- Suppressing failed attempts; document what was tried and why it failed, as this guides future work
- Conflating computational evidence with proof -- numerical checks support but never replace logical argument

## Error Handling

| Failure Mode | Detection | Recovery |
|---|---|---|
| **Hallucinated citation** | Verification pass finds a cited theorem that does not exist or says something different | Remove the citation; either find the correct source via search or prove the claimed result from scratch |
| **Specification gaming** | Problem has multiple interpretations and the agent solves the trivial one | Re-read the problem statement carefully; ask the user to clarify if genuinely ambiguous; default to the hardest reasonable interpretation |
| **Circular reasoning** | Verification pass finds the proof assumes what it is trying to show, possibly in disguised form | Identify the exact point of circularity; restructure the argument to avoid it; often requires a fundamentally different approach |
| **Gap in case analysis** | Verification identifies missing cases | Enumerate all cases systematically; prove each one; if a case resists proof, flag it as the critical obstacle |
| **Divergent revision loop** | 3+ revision cycles without convergence; each fix introduces new errors | Stop revising the current approach; step back and try a different proof strategy entirely |
| **Problem beyond capability** | Repeated failures across multiple strategies; problem requires techniques not in the agent's repertoire | Admit failure honestly; report partial results and identify the specific barrier; suggest relevant literature or techniques the user might pursue |

## Limitations

- **Cannot replace human mathematical judgment for novel research.** The generate-verify-revise loop catches many errors but not all subtle ones. In Aletheia's Erdos evaluation, only 6.5% of candidate solutions were both correct and meaningful -- human expert review remains essential.
- **Literature search is imperfect.** The agent may miss relevant papers, especially recent preprints or results in adjacent fields using different terminology.
- **Verification is not formal proof checking.** The verification pass uses natural-language reasoning, not a formal proof assistant (Lean, Coq, Isabelle). It catches logical gaps but cannot guarantee correctness.
- **Scales poorly with proof length.** Very long proofs (50+ steps) accumulate errors across steps. For such proofs, decompose into independently verifiable lemmas.
- **Sensitive to problem formalization.** Ambiguous or imprecise problem statements lead to specification gaming. The agent works best when the problem is stated precisely in standard mathematical notation.
- **Computational experiments are bounded.** Can only test small cases; extrapolation from small cases to general results is heuristic, not rigorous.

## Reference

**Paper**: Feng, T., Trinh, T.H., Bingham, G., Hwang, D., & Chervonyi, Y. (2026). *Towards Autonomous Mathematics Research.* arXiv:2602.10177v1. [https://arxiv.org/abs/2602.10177v1](https://arxiv.org/abs/2602.10177v1)

**Key takeaway**: The generate-verify-revise loop with decoupled verification, aggressive tool use, and honest failure admission is the core pattern. Look at Sections 3-5 for the inference-time scaling law, Section 6 for the Erdos evaluation methodology, and the appendices for all prompts used.

**Code and prompts**: [https://github.com/google-deepmind/superhuman/tree/main/aletheia](https://github.com/google-deepmind/superhuman/tree/main/aletheia)