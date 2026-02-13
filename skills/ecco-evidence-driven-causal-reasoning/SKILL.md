---
name: ecco-evidence-driven-causal-reasoning
description: >
  Apply evidence-driven causal reasoning to compiler optimization pass selection and ordering.
  Uses the ECCO framework: analyze static code features, build causal explanations linking
  features to pass effectiveness, then guide search-based optimization with LLM-generated
  optimization intents. Triggers: "optimize compiler passes", "find best LLVM pass order",
  "reduce execution cycles for this code", "causal compiler optimization",
  "why does this optimization pass help", "tune LLVM pass sequence for performance".
---

This skill enables Claude to apply the ECCO (Evidence-driven Causal Reasoning for Compiler Optimization) framework to analyze C/C++ code, identify static features that causally determine which compiler optimization passes are beneficial, and produce reasoned pass sequences that outperform default `-O3`. Rather than guessing pass orderings or blindly searching, Claude constructs explicit causal chains from code structure to optimization decisions, then uses those chains to guide targeted search over pass orderings.

## When to Use

- When a user asks to optimize LLVM pass ordering for a specific C/C++ program or module
- When a user wants to understand *why* certain compiler passes help or hurt a given piece of code
- When a user is tuning compilation flags beyond `-O2`/`-O3` and wants data-driven pass selection
- When a user has a performance-critical inner loop and wants to reason about which transformations (unrolling, vectorization, inlining) are causally justified by the code's structure
- When a user is building an auto-tuning pipeline and needs an LLM-guided search strategy instead of random or exhaustive search
- When a user asks to reduce execution cycles, code size, or compilation time through intelligent pass sequencing

## Key Technique

**The problem with black-box search and naive LLM approaches.** Traditional compiler auto-tuning (random search, genetic algorithms, Bayesian optimization) explores pass orderings without understanding the code. LLMs applied naively to pass selection tend to pattern-match on surface syntax rather than reason about *why* a pass helps. ECCO resolves this by constructing explicit causal evidence: "this code has deeply nested loops with invariant bounds and no aliasing, therefore loop unrolling followed by vectorization will reduce cycles because the trip count is statically known and SIMD lanes can be filled."

**Reverse-engineered Chain-of-Thought.** ECCO builds training data by working backwards: given a program and a known-beneficial pass sequence, it extracts the static code features (loop depth, branch density, memory access patterns, function call frequency, data types) that *explain* why those passes succeeded. This produces Chain-of-Thought examples of the form: "Feature: loop with constant trip count of 256 and no carried dependencies. Evidence: `loop-unroll` reduced dynamic instruction count by 4x on profiling. Conclusion: apply `loop-unroll` with factor 8 before `slp-vectorizer`." The model learns causal rules, not brittle sequences.

**Collaborative LLM + Genetic Algorithm inference.** At optimization time, the LLM analyzes new code and emits *optimization intents* -- structured descriptions of which transformations to apply and why. These intents constrain the mutation operators of a genetic algorithm: instead of randomly inserting/removing/reordering passes, the GA mutates within the subspace the LLM identified as causally justified. The GA evaluates candidates by actually compiling and profiling, feeding results back so the LLM can refine its causal model. This achieves an average 24.44% cycle reduction over `-O3` across seven benchmark suites.

## Step-by-Step Workflow

1. **Extract LLVM IR from the target code.** Compile the source with `clang -S -emit-llvm -O0 -o output.ll input.c` to get unoptimized IR. This is the representation ECCO reasons over, since IR exposes the features that passes act on.

2. **Identify static code features from the IR.** Analyze the IR for: loop nesting depth, trip counts (constant vs. dynamic), branch density, memory access patterns (stride, aliasing via `noalias`/`restrict`), function call sites and sizes (inlining candidates), data types (integer width, float/double, vector types), and def-use chain length. Use `opt -analyze -loops -scalar-evolution -basicaa` or manual inspection.

3. **Build causal feature-to-pass mappings.** For each identified feature, state the causal hypothesis:
   - Constant trip count + no loop-carried dependency → `loop-unroll` beneficial
   - Stride-1 memory access + compatible data width → `slp-vectorizer` beneficial
   - Small callee with single call site → `inline` beneficial, enables further interprocedural opts
   - Redundant loads across branches → `gvn` (global value numbering) beneficial
   - Dead stores after transformation → `dse` (dead store elimination) as cleanup pass

4. **Generate an optimization intent document.** Write a structured plan stating: which passes to apply, in what order, and the causal evidence for each. Order passes so enabling passes come first (e.g., `inline` before `gvn`, `loop-unroll` before `slp-vectorizer`, transformations before cleanup passes like `dce` and `dse`).

5. **Construct a seed pass sequence from the intent.** Translate the optimization intent into an `opt` command: `opt -passes='inline,gvn,loop-unroll,slp-vectorizer,dce' input.ll -o optimized.bc`. This is the causally-grounded seed.

6. **Define constrained mutation operators for search.** If further tuning is needed, define mutation rules that respect causal constraints: (a) reorder passes only within the same phase (e.g., swap two cleanup passes, but don't move cleanup before transformation), (b) adjust pass parameters (unroll factor, inline threshold), (c) add/remove passes only if a causal justification exists from step 3.

7. **Evaluate candidates by compiling and profiling.** For each candidate sequence: compile with `opt -passes='...' | llc -o output.o`, link, and measure execution cycles with `perf stat` or hardware counters. Record instruction count, branch misses, cache misses alongside cycles.

8. **Feed profiling evidence back into reasoning.** If a candidate underperforms, diagnose why: did unrolling increase code size enough to cause instruction cache misses? Did inlining increase register pressure? Update the causal model and regenerate intents accordingly.

9. **Iterate until convergence or budget exhaustion.** Repeat steps 6-8 for a fixed number of generations (typically 10-50). The causal constraints dramatically reduce the search space compared to unconstrained GA, so convergence is fast.

10. **Output the final pass sequence with causal justification.** Deliver both the optimized pass ordering and the reasoning chain explaining each decision, so the user understands *why* this sequence works for their code.

## Concrete Examples

**Example 1: Optimizing a matrix multiplication kernel**

```
User: "Optimize the compiler passes for this matrix multiply in C. It's the hot loop in my program."

void matmul(float *A, float *B, float *C, int N) {
    for (int i = 0; i < N; i++)
        for (int j = 0; j < N; j++)
            for (int k = 0; k < N; k++)
                C[i*N+j] += A[i*N+k] * B[k*N+j];
}

Approach:
1. Extract IR: clang -S -emit-llvm -O0 -o matmul.ll matmul.c
2. Identify features:
   - Triple-nested loop, dynamic trip count (N), no early exits
   - Stride-1 access on A (inner k), stride-N access on B (poor locality)
   - Float multiply-accumulate pattern (FMA candidate)
   - No aliasing between A, B, C (add restrict qualifiers or use -fno-strict-aliasing)
3. Causal mappings:
   - FMA pattern → enable `-passes=float2int,instcombine` won't help (already float), but
     `slp-vectorizer` can pack 4/8 float ops. Causal: stride-1 on A enables vectorization.
   - Loop interchange (swap j,k) would make B stride-1. Causal: cache miss reduction.
   - After interchange, inner loop has stride-1 on both A row and B column → `loop-unroll`
     with factor 4 fills vector registers.
   - `licm` (loop-invariant code motion) hoists `i*N` computation out of inner loops.
4. Recommended pass sequence:
   opt -passes='sroa,licm,loop-interchange,loop-unroll{O3},slp-vectorizer,instcombine,dce'

Output:
Pass sequence: sroa → licm → loop-interchange → loop-unroll → slp-vectorizer → instcombine → dce

Causal justification:
- sroa: Promote stack allocations to registers for clean SSA form
- licm: Hoist i*N multiply out of j/k loops (invariant in inner loops)
- loop-interchange: Swap j and k loops so B access becomes stride-1 (reduces cache misses)
- loop-unroll: Unroll inner loop to expose ILP and fill vector registers
- slp-vectorizer: Pack adjacent float operations into SIMD instructions (now possible
  because both A and B have stride-1 access after interchange)
- instcombine + dce: Clean up redundant instructions from transformations
```

**Example 2: Analyzing why a pass hurts performance**

```
User: "I added -loop-unroll to my pipeline but my code got slower. Why?"

void process(int *data, int n, int *lookup) {
    for (int i = 0; i < n; i++) {
        int idx = data[i];
        if (idx >= 0 && idx < 1024)
            data[i] = lookup[idx];
    }
}

Approach:
1. Identify features:
   - Single loop, dynamic trip count, data-dependent branch inside
   - Indirect memory access: lookup[data[i]] — unpredictable access pattern
   - Branch depends on runtime values (not statically predictable)
2. Causal analysis of why unrolling hurts:
   - Unrolling replicates the branch and indirect load. The indirect access pattern
     means each iteration may cache-miss independently.
   - Unrolling increases code size (replicated branch + load) without reducing
     loop overhead proportionally, because the bottleneck is memory latency, not
     loop control.
   - Larger unrolled body may evict useful instruction cache lines.
3. Evidence: measure with perf stat — expect higher L1-icache-load-misses and
   similar or higher L1-dcache-load-misses after unrolling.

Output:
Diagnosis: Loop unrolling is counterproductive here because:
1. The loop is memory-bound (indirect lookup[data[i]]), not compute-bound
2. Unrolling replicates cache-missing loads without hiding latency
3. Increased code size pressures instruction cache

Recommendation: Remove loop-unroll for this function. Instead, consider:
- prefetch: Insert software prefetch for lookup[data[i+k]] to hide memory latency
- Profile-guided optimization (PGO) to let the compiler see the branch bias
Pass sequence: licm,simplifycfg,loop-simplify (keep the loop tight, reduce overhead)
```

**Example 3: Building an auto-tuning pipeline with causal constraints**

```
User: "I want to auto-tune pass ordering for my embedded DSP code. How should I set up
the search?"

Approach:
1. Analyze the DSP code's dominant features:
   - Fixed-point arithmetic (integer multiply-accumulate)
   - Small, tight loops with known trip counts (filter taps = 64)
   - No dynamic allocation, no function pointers
   - Circular buffer access patterns
2. Define the causally-justified pass pool (exclude irrelevant passes):
   - Include: loop-unroll, slp-vectorizer, instcombine, licm, dse, dce, gvn
   - Exclude: loop-interchange (single loop), inline (no calls in hot path),
     loop-distribute (no cross-iteration deps to break)
3. Define ordering constraints from causal analysis:
   - licm must precede loop-unroll (hoist invariants before replication)
   - loop-unroll must precede slp-vectorizer (expose parallel operations)
   - gvn must precede dse (find redundancies before eliminating stores)
   - instcombine and dce are cleanup — always last
4. Configure GA with these constraints:
   - Chromosome: permutation of {licm, gvn, loop-unroll, slp-vectorizer, dse}
     + fixed suffix {instcombine, dce}
   - Mutation: swap two adjacent non-constrained passes
   - Crossover: order crossover preserving constraint edges
   - Fitness: execution cycles measured via hardware timer on target DSP
5. Run 20 generations with population 30 — small search space due to constraints.

Output:
Auto-tuning configuration:
- Pass pool: 7 passes (vs. 50+ in unconstrained search)
- Ordering constraints: 4 causal edges (reduces permutations from 5040 to ~30)
- Expected convergence: 10-20 generations (vs. 100+ unconstrained)
- Measurement: cycle count on target hardware via cross-compilation + execution
```

## Best Practices

- **Do:** Always extract IR at `-O0` for analysis, then apply passes explicitly. Starting from `-O2` IR hides features the passes already transformed.
- **Do:** State causal evidence for every pass in the sequence. "I added `gvn` because there are redundant loads across the if/else branches at lines 12-18" is actionable. "I added `gvn` because it usually helps" is not.
- **Do:** Order passes in phases: canonicalization (sroa, mem2reg) → analysis-enabling (inline, licm) → transformation (unroll, vectorize, interchange) → cleanup (instcombine, dce, dse). Causal reasoning should respect this phase structure.
- **Do:** Profile before and after with hardware counters (`perf stat -e cycles,instructions,cache-misses,branch-misses`) to verify that the causal hypothesis holds.
- **Avoid:** Applying loop unrolling to memory-bound loops with indirect or random access patterns. The causal model must account for the memory hierarchy, not just instruction count.
- **Avoid:** Including passes in the search space that have no causal connection to the code's features. Every pass in the pool should be justified by at least one feature in the IR.

## Error Handling

- **Pass ordering crashes LLVM:** Some pass orderings are invalid (e.g., running a function pass that expects canonical loop form before loop-simplify). Always include `loop-simplify` and `lcssa` before loop transformation passes. If `opt` segfaults, check that canonicalization passes precede transformation passes.
- **Profiling noise masks signal:** Cycle measurements vary between runs. Take the median of 5+ runs. For improvements under 5%, use statistical significance testing (paired t-test) before concluding a pass sequence is better.
- **Causal hypothesis is wrong:** If profiling contradicts the expected improvement (e.g., vectorization slowed code down), check for: (a) short trip counts where vector setup overhead dominates, (b) unaligned memory access causing fallback to scalar, (c) register pressure increase causing spills. Update the causal model and re-derive the pass sequence.
- **Code changes invalidate the sequence:** Pass orderings are tuned to specific code structure. If the source code changes significantly (new loops, different data types), re-run the analysis from step 1. Minor changes (constant tweaks, added logging) usually don't affect the optimal sequence.

## Limitations

- **Requires compilation and profiling infrastructure.** The feedback loop depends on actually compiling and running the code. Cross-compilation targets without execution capability (e.g., embedded targets without emulators) limit the approach to static reasoning only.
- **Dynamic behavior not captured.** The causal model reasons over static IR features. Input-dependent behavior (e.g., branch bias that changes with data) requires profile-guided optimization data to supplement the static analysis.
- **LLVM-specific.** The pass names, ordering rules, and IR analysis are specific to the LLVM toolchain. GCC, MSVC, and other compilers have different pass infrastructures. The causal reasoning methodology transfers, but the specific pass mappings do not.
- **LLM reasoning is approximate.** Claude's causal analysis is a best-effort heuristic, not a formal proof. Always validate with profiling. The value is in dramatically narrowing the search space, not in guaranteeing optimal results from reasoning alone.
- **Diminishing returns past `-O3`.** For code that `-O3` already optimizes well, the additional gain from custom pass ordering may be small (single-digit percentage). The largest gains come from code with unusual structure that generic heuristics handle poorly.

## Reference

[ECCO: Evidence-Driven Causal Reasoning for Compiler Optimization](https://arxiv.org/abs/2602.00087v1) (Pan et al., 2026). Key sections: the reverse-engineering methodology for Chain-of-Thought dataset construction (how to map code features to pass evidence), and the collaborative LLM-GA inference mechanism (how causal intents constrain genetic search). The paper reports 24.44% average cycle reduction over `-O3` across CBench, MiBench, and five other benchmark suites.