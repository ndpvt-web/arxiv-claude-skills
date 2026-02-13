---
name: "reducing-costs-proof-synthesis"
description: "Generate formally verified Rust code with Verus specifications and proofs using the VeruSyn methodology. Applies self-synthesis, tutorial-based synthesis, and chain-of-thought debugging to produce correct-by-construction Rust systems code. Trigger phrases: 'verify this Rust code', 'add Verus proofs', 'write a formal specification', 'prove this function correct', 'generate verified Rust', 'add requires and ensures clauses'."
---

This skill enables Claude to generate Rust code with formal correctness proofs using the Verus verification framework, following the VeruSyn methodology from "Reducing the Costs of Proof Synthesis on Rust Systems by Scaling Up a Seed Training Set" (Di et al., 2026). It teaches Claude to write Verus-annotated Rust with preconditions (`requires`), postconditions (`ensures`), loop invariants, termination proofs (`decreases`), ghost variables, and proof blocks — then iteratively debug verification failures using structured chain-of-thought reasoning.

## When to Use

- When the user asks to write Rust code that must be formally verified for correctness
- When adding Verus annotations (specifications and proofs) to existing Rust functions
- When the user wants to prove properties about data structures (sorted, no duplicates, bounded)
- When writing systems-level Rust (memory allocators, concurrent data structures, OS components) that needs safety guarantees
- When translating informal correctness arguments into machine-checked Verus proofs
- When debugging Verus verification errors by analyzing error messages and adding missing proof obligations
- When the user asks to specify loop invariants or termination metrics for recursive/iterative Rust code

## Key Technique

VeruSyn addresses the fundamental bottleneck in verified code generation: LLMs have orders of magnitude less verified-code training data compared to regular code. The paper's core insight is a three-stage data synthesis pipeline that scales a small seed set of ~10,000 verified Verus programs into 6.9 million verified programs, each with a formal specification and a proof that meets it.

The pipeline has three synthesis strategies. **Self-synthesis** fine-tunes a model on the seed set, then uses it to generate new programs via continuation prompts, filtering through Verus verification and SimHash deduplication (terminating when 95% of a batch are duplicates). **Tutorial-based synthesis** uses ~1,000 expert-written seed programs mapped to Verus tutorial chapters, generating thousands of variants per seed to ensure coverage of all 20 tracked Verus features (invariants, quantifiers, nonlinear arithmetic, ghost variables, etc.). **Agent trajectory synthesis** captures complete debugging sessions — the reasoning steps and code edits an agent performs when iterating on a failing proof — producing chain-of-thought training data that teaches models *how to debug* verification failures, not just produce correct proofs on the first attempt.

The practical takeaway for Claude: when generating Verus proofs, follow a structured workflow of specification-first design, proof sketch generation, iterative debugging against Verus error output, and explicit anti-cheating discipline (never use `assume`, `admit`, or `assert(false)` to bypass verification). Always include `decreases` clauses for loops and recursion, and use quantified assertions in proof blocks to guide the solver.

## Step-by-Step Workflow

1. **Analyze the target Rust function** — identify inputs, outputs, side effects, and the correctness property the user wants proven (e.g., "the output is sorted," "no out-of-bounds access," "the invariant is preserved").

2. **Write the specification first** — express the correctness property as `requires` (preconditions on inputs) and `ensures` (postconditions on the return value and any mutable references) clauses. Use `old(self)` for mutable reference pre-state.

```rust
fn binary_search(v: &Vec<u64>, key: u64) -> (result: usize)
    requires
        forall|i: int, j: int| 0 <= i < j < v@.len() ==> v@[i] <= v@[j],
    ensures
        result <= v.len(),
        result > 0 ==> v@[(result - 1) as int] < key,
        result < v.len() ==> key <= v@[result as int],
```

3. **Define spec functions for abstract views** — use `pub closed spec fn view` and the `@` operator to bridge between executable types and their logical representations (e.g., `self.v@` gives the abstract `Seq` view of a `Vec`).

4. **Add loop invariants and termination proofs** — every loop needs an `invariant` clause stating what holds at each iteration, and a `decreases` clause proving the loop terminates. Every recursive function needs a `decreases` clause.

```rust
while lo < hi
    invariant
        0 <= lo <= hi <= v.len(),
        lo > 0 ==> v@[(lo - 1) as int] < key,
        hi < v.len() ==> key <= v@[hi as int],
    decreases hi - lo,
```

5. **Insert proof blocks for non-obvious steps** — when the SMT solver cannot automatically verify a step, add `proof { ... }` blocks with quantified assertions (`assert forall`, `assert exists`) or lemma calls to guide it.

6. **Run Verus verification** — execute `verus --crate-type lib <file.rs>` and capture the output. Verus reports errors with specific spans and obligation names.

7. **Debug verification failures using structured CoT reasoning** — for each error: (a) identify which specification clause failed, (b) determine whether the issue is a missing invariant, an insufficient assertion, or an incorrect spec, (c) formulate the fix, (d) apply it, (e) re-verify. Chain these steps explicitly.

8. **Validate proof integrity** — check that no `assume`, `admit`, `assert(false)`, `external_body`, or `axiom` annotations were introduced. Ensure function signatures (parameters, return types, requires/ensures) were not weakened from the user's original intent.

9. **Verify feature completeness** — confirm that all relevant Verus constructs are used correctly: `ghost` variables for verification-only state, `tracked` for linear resources, `choose` for witness extraction, `trigger` annotations for quantifier instantiation.

10. **Present the verified code** with clear separation of specification, proof, and executable code, explaining the key proof obligations and how each invariant/assertion contributes to the overall correctness argument.

## Concrete Examples

**Example 1: Verified insertion into a sorted vector**

User: "Write a Verus-verified function that inserts an element into a sorted vector while maintaining the sorted property."

Approach:
1. Define a `sorted` spec function over `Seq<u64>`
2. Specify requires (input is sorted) and ensures (output is sorted, contains new element, length increased by 1)
3. Use a while loop with invariant tracking the insertion point search
4. Add proof block showing elements remain ordered after insertion

Output:
```rust
use vstd::prelude::*;

verus! {

pub open spec fn sorted(s: Seq<u64>) -> bool {
    forall|i: int, j: int| 0 <= i < j < s.len() ==> s[i] <= s[j]
}

fn sorted_insert(v: &mut Vec<u64>, elem: u64)
    requires
        sorted(old(v)@),
    ensures
        sorted(v@),
        v@.len() == old(v)@.len() + 1,
        exists|i: int| 0 <= i < v@.len() && v@[i] == elem,
{
    let mut idx: usize = 0;
    while idx < v.len()
        invariant
            0 <= idx <= v.len(),
            sorted(v@),
            v@ == old(v)@,
            forall|j: int| 0 <= j < idx as int ==> v@[j] <= elem,
        decreases v.len() - idx,
    {
        if v[idx] > elem {
            break;
        }
        idx = idx + 1;
    }
    v.insert(idx, elem);
    proof {
        assert forall|i: int, j: int|
            0 <= i < j < v@.len() implies v@[i] <= v@[j]
        by {
            // Elements before idx: sorted by invariant, all <= elem
            // elem at idx: >= all before, <= all after
            // Elements after idx: shifted from old, still sorted
        }
    }
}

} // verus!
```

**Example 2: Verified binary search with proof of correctness**

User: "Prove that a binary search on a sorted array returns the correct insertion point."

Approach:
1. Specify precondition: input slice is sorted
2. Specify postcondition: returned index is the correct partition point
3. Add loop invariants maintaining the search window bounds
4. Include decreases clause on `hi - lo`

Output:
```rust
use vstd::prelude::*;

verus! {

fn binary_search(v: &Vec<u64>, key: u64) -> (idx: usize)
    requires
        forall|i: int, j: int| 0 <= i < j < v@.len() ==> v@[i] <= v@[j],
    ensures
        0 <= idx <= v.len(),
        forall|j: int| 0 <= j < idx as int ==> v@[j] < key,
        forall|j: int| idx as int <= j < v@.len() as int ==> key <= v@[j],
{
    let mut lo: usize = 0;
    let mut hi: usize = v.len();
    while lo < hi
        invariant
            0 <= lo <= hi <= v.len(),
            forall|j: int| 0 <= j < lo as int ==> v@[j] < key,
            forall|j: int| hi as int <= j < v@.len() as int ==> key <= v@[j],
            forall|i: int, j: int| 0 <= i < j < v@.len() ==> v@[i] <= v@[j],
        decreases hi - lo,
    {
        let mid = lo + (hi - lo) / 2;
        if v[mid] < key {
            lo = mid + 1;
        } else {
            hi = mid;
        }
    }
    lo
}

} // verus!
```

**Example 3: Debugging a failing Verus proof via chain-of-thought**

User: "Verus reports `postcondition not satisfied` on my `remove` function for a sorted vec. Help me fix it."

Approach:
1. Read the Verus error output identifying which ensures clause failed
2. Trace the proof obligation: after removal, the vector must still be sorted
3. Identify the missing assertion: Verus needs help seeing that removing an element from a sorted sequence preserves sortedness
4. Add a proof block with a quantified assertion bridging the gap

Debugging CoT:
```
Error: postcondition `sorted(self@)` not satisfied after self.v.remove(i)

Reasoning: After removing index i, elements at indices [0, i) are unchanged
and elements at indices [i, new_len) were previously at [i+1, old_len).
The solver needs an explicit assertion that the subsequences remain ordered
and that the boundary between them is also ordered.

Fix: Add proof block asserting the element-wise ordering is preserved:

proof {
    assert(forall|j: int, k: int|
        0 <= j < k < self@.len() ==> self@[j] <= self@[k])
    by {
        // For j, k both < i: same as old, sorted
        // For j, k both >= i: were at j+1, k+1 in old, sorted
        // For j < i, k >= i: old[j] <= old[i] <= old[k+1] = new[k]
    }
}
```

## Best Practices

**Do:**
- Always write `decreases` clauses for every loop and every recursive function — Verus requires them for termination proofs, and omitting them is a common source of errors
- Use `old(self)` in mutable method postconditions to refer to the pre-call state
- Start with the specification before writing any proof — getting the spec right makes the proof dramatically easier
- Use `assert forall ... by { ... }` blocks to guide the SMT solver when automatic verification times out
- Keep proof blocks minimal and close to the code they justify — each assertion should address one specific solver obligation

**Avoid:**
- Never insert `assume(false)`, `admit()`, `assert(false)`, `external_body`, or `axiom` to make verification pass — these defeat the purpose of verification and are classified as cheating
- Do not weaken the user's original specification to make proofs easier — the spec defines what correctness means
- Avoid overly complex quantifier triggers that cause solver instability — prefer simple trigger patterns and add explicit `#[trigger]` annotations when needed
- Do not conflate spec, proof, and exec code — spec functions cannot have side effects, proof functions cannot affect execution, and exec functions should minimize inline proof logic

## Error Handling

| Verus Error | Likely Cause | Fix Strategy |
|---|---|---|
| `postcondition not satisfied` | Missing proof step between code and ensures clause | Add `proof { assert(...) }` block bridging the logical gap |
| `invariant not satisfied at loop entry` | Invariant is too strong for initial state | Weaken invariant or add precondition establishing it before the loop |
| `invariant not maintained` | Loop body breaks the invariant | Add assertions inside the loop body showing the invariant is restored |
| `decreases not decreasing` | Termination metric does not strictly decrease | Fix the decreases expression or add a `when` guard for the base case |
| `arithmetic overflow` | Verus defaults to checked arithmetic | Use `as int` casts in specifications or add bounds to requires clauses |
| `cannot verify trigger` | Quantifier instantiation issue | Add explicit `#[trigger]` annotations or restructure the quantifier |
| Solver timeout | Proof too complex for automatic verification | Break the proof into smaller lemma functions and call them sequentially |

When Verus fails, follow the VeruSyn debugging pattern: read the error, identify the failing obligation, formulate a hypothesis about what's missing, add the minimal assertion to close the gap, and re-verify. Repeat until all obligations pass.

## Limitations

- Verus is specific to Rust — this approach does not apply to other languages without equivalent verification frameworks (Dafny, F*, etc. have their own ecosystems)
- Complex systems proofs (concurrent data structures, distributed protocols) may require manual lemma libraries that go beyond what can be synthesized automatically
- Verus's SMT solver has inherent scalability limits; functions exceeding ~200 lines of proof may need manual decomposition into smaller lemmas
- Nonlinear arithmetic reasoning (multiplication, division of symbolic values) is a known weak point of SMT solvers and may require manual `nonlinear_arith` proof blocks
- The VeruSyn methodology is optimized for Verus-specific syntax and idioms — transferring proof strategies to other verification tools requires adaptation
- Ghost/tracked variable reasoning for linear types and resource management is an advanced Verus feature that may require domain-specific expertise beyond what this workflow covers

## Reference

**Paper:** Di, N., Chen, T., Lu, S., Lu, S., & Gong, Y. (2026). "Reducing the Costs of Proof Synthesis on Rust Systems by Scaling Up a Seed Training Set." arXiv:2602.04910v1. [https://arxiv.org/abs/2602.04910v1](https://arxiv.org/abs/2602.04910v1)

Look for: The three-stage synthesis pipeline (self-synthesis, tutorial-based, agent trajectory), the anti-cheating validation checks in Appendix B, and the evaluation on VeruSAGE-Bench showing 47x cost reduction versus Claude Sonnet 4.5 at comparable accuracy. The Verus tutorial coverage matrix (Table 2) maps 20 feature keywords to their tutorial chapters — use it as a checklist when writing proofs.