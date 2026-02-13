---
name: "mathliblemma-folklore-lemma-generation"
description: "Multi-agent system for discovering and formalizing missing 'folklore' lemmas in Lean 4 / Mathlib. Identifies gaps in formal math libraries, generates Lean 4 statements, type-checks them, and iterates until verified. Trigger phrases: 'find missing lemmas in Mathlib', 'generate folklore lemma', 'formalize lemma in Lean 4', 'Mathlib gap analysis', 'discover missing Lean theorems', 'automate Lean formalization'."
---

# MathlibLemma: Folklore Lemma Discovery and Formalization for Lean 4

This skill enables Claude to operate as a multi-agent pipeline that discovers missing "folklore" lemmas -- well-known mathematical results that mathematicians use routinely but that have not yet been formalized in Lean 4's Mathlib library. Drawing from the MathlibLemma framework (Liu et al., 2026), the technique decomposes the problem into four coordinated phases: gap identification, natural-language statement drafting, Lean 4 formalization, and iterative type-check verification. The result is a stream of verified, merge-ready Lean 4 lemma declarations that fill the connective tissue between existing Mathlib theorems.

## When to Use

- When a user is working in Lean 4 and hits a missing lemma that "should obviously exist" in Mathlib but doesn't.
- When a user wants to systematically audit a Mathlib module (e.g., `Mathlib.Topology.Basic`) for commonly-needed but absent intermediate results.
- When a user asks to formalize a well-known mathematical fact (from a textbook, paper, or informal proof) into a type-checked Lean 4 statement with proof.
- When building a verified library of supplementary lemmas for a specific mathematical domain (algebra, analysis, topology, combinatorics, number theory).
- When a user wants to contribute new lemmas to Mathlib and needs help identifying what is actually missing and how to formalize it to Mathlib style standards.
- When a user has a Lean 4 proof that fails because an intermediate step has no Mathlib support, and they need that bridge lemma generated.

## Key Technique

**Gap-Finding via Proof Dependency Analysis.** The core insight of MathlibLemma is that missing lemmas can be discovered systematically rather than stumbled upon. The Gap Finder agent analyzes the dependency graph of existing Mathlib theorems and identifies structural holes: pairs of theorems where a plausible intermediate result is missing, API patterns where one direction of an iff exists but not the other, coercions or simp lemmas that are conspicuously absent for a given type, and theorems that exist for one algebraic structure (e.g., `Group`) but not an analogous one (e.g., `AddGroup`). The agent also cross-references informal mathematical sources (textbooks, MathOverflow, Wikipedia) against the Mathlib namespace to spot well-known results that lack formal counterparts.

**Formalization with Iterative Type-Check Feedback.** Once a candidate lemma is identified in natural language, the Formalizer agent translates it into a Lean 4 `theorem` or `lemma` declaration using Mathlib's existing type universe, notation, and naming conventions. This is not a one-shot process. The Verifier agent runs `lean --run` (or `lake build`) to type-check the statement. If it fails, the error message -- including unknown identifiers, type mismatches, or universe issues -- is fed back to the Formalizer for correction. This loop typically converges in 2-5 iterations. The key is that Lean's type checker provides precise, actionable error signals that guide the LLM toward correct formalization far more effectively than natural-language feedback alone.

**Quality Gate: Mathlib Style Compliance.** A generated lemma is not merely type-correct; it must also follow Mathlib's strict style conventions -- correct `namespace`, dot-notation compatibility, `@[simp]` annotations where appropriate, and proofs that use the preferred tactic style (e.g., `exact`, `simp`, `ring`, `linarith` over manual term-mode). The Refinement agent checks these conventions and rewrites proofs to be idiomatic before declaring a lemma ready for contribution.

## Step-by-Step Workflow

1. **Scope the target domain.** Identify the Mathlib module or mathematical area to audit (e.g., `Mathlib.Analysis.SpecificLimits.Basic` or "basic properties of cyclic groups"). Use `lake env printPaths` and grep through `.lean` files to inventory existing declarations in the target namespace.

2. **Run gap-finding heuristics.** For the target namespace, apply these concrete checks:
   - **Symmetry gaps:** For each `theorem foo_bar`, check whether `bar_foo` exists. For each `A → B` implication, check whether `B → A` or `A ↔ B` exists.
   - **Algebraic analogy gaps:** For each lemma about `Mul`/`Group`/`Ring`, check whether the additive counterpart (`Add`/`AddGroup`/`AddRing`) exists (and vice versa via the `@[to_additive]` attribute).
   - **Simp completeness:** Identify terms that appear in `simp` lemma LHS patterns and check that all natural rewrite directions are covered.
   - **Coercion chains:** Check for missing coercion lemmas between related types (e.g., `Nat → Int → Rat → Real`).
   - **Textbook cross-reference:** Compare the namespace's coverage against a standard reference (e.g., Bourbaki, Lang's Algebra, Rudin's Principles) and flag missing standard results.

3. **Draft candidate lemma statements in natural language.** For each identified gap, write a precise English statement: "For any commutative ring R and elements a, b in R, if a divides b and b divides a, then a and b are associates." Include the expected Lean 4 types and the Mathlib namespace where it belongs.

4. **Formalize into Lean 4.** Translate each natural language statement into a Lean 4 declaration. Use Mathlib's existing definitions, type classes, and naming conventions. Skeleton:
   ```lean
   import Mathlib.RingTheory.Associated

   theorem dvd_dvd_iff_associated {R : Type*} [CommMonoid R]
       (a b : R) (h1 : a ∣ b) (h2 : b ∣ a) : Associated a b :=
     ⟨IsUnit.mk0 _ (by sorry), by sorry⟩
   ```
   Use `sorry` for proof bodies initially -- the priority is a type-correct statement.

5. **Type-check the statement.** Run the Lean 4 type checker on the file. Parse error output for:
   - `unknown identifier`: wrong import or misspelled name -- search Mathlib for the correct name.
   - `type mismatch`: wrong type class assumption or argument order -- adjust the signature.
   - `universe inconsistency`: add explicit universe annotations.
   Feed each error back into the formalization and re-check. Iterate until the statement (with `sorry` proofs) compiles.

6. **Fill in the proof.** Replace `sorry` with actual tactic proofs. Prefer Mathlib-idiomatic tactics:
   - `exact`, `apply`, `constructor` for structural steps.
   - `simp`, `ring`, `linarith`, `omega`, `norm_num` for computational closure.
   - `rcases`, `obtain` for destructuring hypotheses.
   Run the full file through `lake build` to verify the proof is complete (no remaining `sorry`).

7. **Apply Mathlib style checks.** Verify:
   - Lemma name follows Mathlib naming convention (lowercase, underscores, describes the statement).
   - Appropriate attributes (`@[simp]`, `@[norm_cast]`, `@[to_additive]`, `@[ext]`) are applied.
   - The lemma is placed in the correct namespace and file.
   - Proof uses preferred idioms (no `have` chains when `calc` or `simp` suffices).

8. **Batch and deduplicate.** Before finalizing, search Mathlib's current HEAD for equivalent statements that may have been added recently (Mathlib evolves daily). Use `grep -r "theorem.*name_fragment"` and `#check` in Lean to confirm the lemma is genuinely new.

9. **Package for contribution.** Format the lemma(s) as a standalone `.lean` file with correct imports, or as a patch to an existing Mathlib file. Include a doc-string explaining what the lemma states and why it is useful.

10. **Validate end-to-end.** Run `lake build` on the full Mathlib project with the new file included. Confirm zero errors and zero warnings. If contributing to Mathlib upstream, run `lake exe lint` and `lake exe checkdecls` as well.

## Concrete Examples

**Example 1: Finding a missing `iff` lemma in order theory**

User: "I'm working in `Mathlib.Order.Basic` and I noticed there's `le_antisymm` but no convenient iff connecting `a ≤ b ∧ b ≤ a` with `a = b` as a single simp lemma. Can you generate it?"

Approach:
1. Search `Mathlib.Order.Basic` for existing `le_antisymm` variants and any `iff` form.
2. Confirm that `le_antisymm_iff` does not exist in current Mathlib.
3. Draft: "For a partial order, `a = b ↔ a ≤ b ∧ b ≤ a`."
4. Formalize:

```lean
import Mathlib.Order.Basic

@[simp]
theorem eq_iff_le_and_ge {α : Type*} [PartialOrder α] {a b : α} :
    a = b ↔ a ≤ b ∧ b ≤ a :=
  ⟨fun h => h ▸ ⟨le_refl _, le_refl _⟩, fun ⟨h1, h2⟩ => le_antisymm h1 h2⟩
```

5. Type-check passes. Verify `@[simp]` orientation is correct (LHS is the more complex side).

Output: A verified, simp-annotated Lean 4 lemma ready for Mathlib contribution.

---

**Example 2: Generating additive counterparts for a multiplicative lemma**

User: "Mathlib has `mul_left_cancel` for groups but I can't find `add_left_cancel` stated as a standalone lemma in `Mathlib.Algebra.Group.Basic`. Can you generate the missing additive version?"

Approach:
1. Locate `mul_left_cancel` in Mathlib and check its signature and attributes.
2. Check whether `@[to_additive]` was already applied (if so, the additive version exists automatically).
3. If not auto-generated, formalize the additive version:

```lean
import Mathlib.Algebra.Group.Basic

@[to_additive]
theorem mul_left_cancel_of_eq {G : Type*} [LeftCancelMonoid G]
    {a b c : G} (h : a * b = a * c) : b = c :=
  LeftCancelMonoid.mul_left_cancel a b c h
```

4. Type-check. If `LeftCancelMonoid` doesn't exist under that name, search for the correct typeclass (`LeftCancelSemigroup`, `CancelMonoid`, etc.) and adjust.
5. Verify the `@[to_additive]` attribute automatically produces `add_left_cancel_of_eq`.

Output: A single declaration that produces both multiplicative and additive versions via Mathlib's `to_additive` machinery.

---

**Example 3: Bridging a proof gap in analysis**

User: "My Lean proof needs the fact that the sum of two continuous functions on a metric space is continuous, but I'm getting errors. Can you find and formalize whatever intermediate lemma I'm missing?"

Approach:
1. Check what the user's proof looks like and identify the exact error.
2. Search `Mathlib.Topology.ContinuousOn` and `Mathlib.Topology.Algebra.Ring.Basic` for `Continuous.add`.
3. If `Continuous.add` exists, the issue is likely a missing import or typeclass. If it genuinely doesn't exist for the user's specific type:
4. Formalize:

```lean
import Mathlib.Topology.Algebra.Ring.Basic

theorem continuous_add_of_continuous {X : Type*} [TopologicalSpace X]
    {M : Type*} [TopologicalSpace M] [Add M] [ContinuousAdd M]
    {f g : X → M} (hf : Continuous f) (hg : Continuous g) :
    Continuous (fun x => f x + g x) :=
  hf.add hg
```

5. Type-check. If `ContinuousAdd` is the wrong class, iterate with the error to find the right one (`TopologicalAddGroup`, `HasContinuousAdd`, etc.).
6. If the lemma already exists under a different name, report the correct name and import to the user instead of generating a duplicate.

Output: Either a new verified bridge lemma or the identification of the correct existing Mathlib lemma with the right import path.

## Best Practices

- **Do:** Always search Mathlib thoroughly before generating a "new" lemma. Use `#check`, `#print`, `exact?`, `apply?`, and grep. Many folklore lemmas exist under non-obvious names.
- **Do:** Use `sorry` strategically -- get the statement to type-check first, then fill proofs. A correct statement with `sorry` is more valuable than a wrong statement with a proof.
- **Do:** Follow Mathlib naming conventions strictly: `theorem dvd_mul_of_dvd_left` not `theorem my_dvd_lemma`. Names should read as the statement itself.
- **Do:** Apply `@[to_additive]` whenever a multiplicative lemma has a natural additive counterpart. This halves the work and keeps the library consistent.
- **Avoid:** Generating lemmas that are trivial consequences of `simp` -- if `simp` closes the goal, a standalone lemma adds clutter, not value.
- **Avoid:** Using `decide` or `native_decide` in proofs intended for Mathlib contribution -- these are brittle and not accepted upstream for non-trivial instances.

## Error Handling

| Error | Cause | Fix |
|-------|-------|-----|
| `unknown identifier 'Foo'` | Wrong import or name changed in recent Mathlib | Search with `grep -r "def Foo\|theorem Foo\|class Foo" ~/.elan/toolchains/` or use `#check Foo` in a Lean file with broad imports |
| `type mismatch: expected α, got β` | Wrong typeclass hierarchy or argument order | Check the actual types with `#check @targetTheorem` and align your signature |
| `universe inconsistency` | Mixing `Type` and `Type*` or `Prop` incorrectly | Add explicit universe variables: `universe u v` and annotate types |
| `tactic 'simp' failed` | Missing simp lemmas in context or wrong goal shape | Try `simp?` to see what lemmas simp tried, then either add `@[simp]` to prerequisites or switch to `exact` |
| `maximum recursion depth exceeded` | Typeclass search loop from conflicting instances | Minimize instances in scope; use `@theorem` (explicit mode) to bypass typeclass resolution |
| Proof compiles but Mathlib linter fails | Style violations (unused variables, wrong attributes) | Run `lake exe lint` and address each warning; common fixes include adding `_` prefixes to unused variables and removing redundant hypotheses |

## Limitations

- **Lean 4 / Mathlib only.** This workflow targets the Lean 4 ecosystem. It does not apply to Coq, Isabelle, or Lean 3 without significant adaptation.
- **Mathlib version sensitivity.** Mathlib changes daily. A generated lemma may type-check against one nightly but fail against the next due to renamed definitions or restructured modules. Always pin your Mathlib version and re-check before submission.
- **Proof search is bounded.** While statement generation succeeds reliably (type-checking is a strong signal), proof generation is harder. Complex proofs requiring novel mathematical insight are beyond the scope of automated generation -- the system excels at "routine" proofs (1-5 tactic steps) not research-level arguments.
- **False positives in gap finding.** Not every "missing" lemma is actually useful. Some gaps exist intentionally because the result is trivially derivable or stylistically disfavored. Always ask: "Would a Mathlib maintainer accept this?"
- **No informal-to-formal for ambiguous statements.** If the natural language statement is mathematically ambiguous (e.g., "continuous" without specifying the topology), the formalization may target the wrong definition. Precision in the natural language input is critical.

## Reference

**Paper:** Liu et al., "MathlibLemma: Folklore Lemma Generation and Benchmark for Formal Mathematics" (arXiv:2602.02561, 2026). Focus on Section 3 (multi-agent pipeline architecture) and Section 4 (gap-finding heuristics and formalization loop) for implementation details. The benchmark of 4,028 type-checked Lean statements provides a useful test set for evaluating formalization quality.