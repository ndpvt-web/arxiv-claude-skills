---
name: "protean-compiler-agile-framework"
description: "Guide fine-grained LLVM compiler phase ordering using the Protean framework's agile optimization approach — clustering passes into subsequences, searching with simulated annealing, and extracting static IR features for ML-driven pass selection. Use when: 'optimize my C compilation beyond -O3', 'find the best LLVM pass order for this code', 'extract compiler IR features for ML', 'speed up this benchmark with custom pass sequences', 'set up Protean compiler phase ordering', 'use ML to pick LLVM optimization passes'."
---

This skill enables Claude to apply the Protean Compiler framework for fine-grained LLVM phase ordering — the practice of finding a program-specific ordering of compiler optimization passes that outperforms fixed sequences like `-O3`. Protean clusters LLVM's 160+ optimization passes into five manageable subsequences (A–E), searches over short "recipes" of these clusters using simulated annealing, and optionally uses a Transformer-based IR2Score model to predict speedup from static IR features. The result is per-module custom pass orderings that achieve up to 4.1% average and 15.7% peak speedup over `-O3` on real benchmarks, with only seconds of extra build time.

## When to Use

- When a user wants to squeeze more performance out of C/C++ code beyond standard `-O3` optimization
- When the user asks how to find the best LLVM pass ordering for a specific program or function
- When building an ML pipeline to predict optimal compiler optimizations from code features
- When the user wants to extract static IR features (instruction mix, loop characteristics, call-graph metrics) for compiler research
- When integrating LLM-based source-to-source optimization with compiler-level phase ordering
- When the user is working on autotuning compiler flags or pass sequences for a benchmark suite
- When exploring how to modify the LLVM compilation pipeline to insert custom optimization decisions

## Key Technique

**The Phase Ordering Problem.** LLVM's `-O3` applies a fixed sequence of ~160 optimization passes. But the optimal ordering depends on the program: loop-heavy code benefits from aggressive vectorization early, while call-heavy code benefits from inlining first. Exhaustive search is impossible (the space is combinatorial and unbounded), so Protean makes it tractable through two innovations.

**Pass Clustering into Subsequences.** Protean uses agglomerative clustering to group LLVM's O3 passes into five clusters (A–E), each representing a coherent optimization phase. Cluster A handles inlining/SROA/function attributes; B handles loop simplification/unrolling/prefetching; C handles GVN/SCCP/BDCE/jump-threading; D handles dead store elimination and dead code elimination; E handles loop distribution/vectorization/load elimination. A "recipe" is a permutation of 0–5 of these clusters (e.g., "CD" means run cluster C then cluster D). This reduces the search space to ~4,000 permutations at max length 5, which captures 97% of achievable speedup.

**Simulated Annealing with IR2Score.** Protean searches the recipe space using simulated annealing with geometric cooling (temperature 100→0). Each candidate recipe is evaluated either by compiling and running the program or by querying IR2Score — a 4-layer Transformer encoder (10 attention heads, feedforward dim 512) trained on 10K C/C++ functions to predict speedup from 141 static IR features. An early-exit condition stops search when the IR stops changing across iterations. The framework also supports a two-step approach: an LLM (e.g., Qwen2.5-32B) performs source-level optimization first, then Protean applies phase ordering on the result, achieving combined speedups of 8–10% on select applications.

## Step-by-Step Workflow

1. **Identify the compilation target.** Determine whether you are optimizing a single file, a module, or a full project. Protean operates at module granularity by default, spawning a subprocess per module to find its optimal recipe independently.

2. **Establish the baseline.** Compile with standard `-O3` and measure execution time on a representative input. This is the baseline that Protean's recipes will be compared against. Use `clang -O3 -o baseline program.c && time ./baseline <input>`.

3. **Enable Protean optimization.** Replace `-O3` with `-OP` and add Protean flags:
   ```
   clang -OP -mllvm -protean \
     -Wprotean,-max-iterations=100 \
     -Wprotean,-use-protean-collect=false \
     program.c -o optimized -lm
   ```
   The `-OP` flag activates Protean's agile driver, which inserts the `ProteanOpt` phase after standard compilation. Set iterations based on time budget: 10 for quick search, 100 for thorough, 500 for exhaustive.

4. **Understand the recipe search space.** Recipes are strings of cluster letters A–E, length 0–5. Examples: `""` (no extra optimization), `"A"` (inlining cluster only), `"CE"` (GVN/SCCP then vectorization), `"ACDBE"` (all five in a specific order). The simulated annealing explores these by swapping, inserting, or removing cluster letters.

5. **Extract static IR features (optional, for ML).** To dump the 141-feature vector for each module/function/loop:
   ```
   clang -OP -mllvm -protean \
     -Wprotean,-protean-output-table \
     program.c -o /dev/null
   ```
   This produces CSV output with columns scoped to module (10 features: FunctionCount, AverageBBPerFunction, TotalBBCount, TotalInstCount, etc.), function (71 features: instruction distributions, loop counts, call metrics), callee/caller (39 features: CallSiteCost, CalleeInstrPerLoop, etc.), and loops (31 features: LoopSize, MaxLoopHeight, StepValueInt, etc.).

6. **Interpret the search output.** Protean logs each iteration's recipe, cost (execution time or IR2Score prediction), temperature, and accept/reject decision. Look for convergence: when the best recipe stabilizes for several consecutive iterations, the search is effectively done.

7. **Apply the winning recipe to production builds.** Once you have the optimal recipe per module (e.g., module1→"CD", module2→"ACE"), encode these into your build system. For a single-file project this is straightforward; for multi-module projects, use per-file compiler flags.

8. **Optionally combine with LLM source-level optimization.** For maximum gains, run an LLM (e.g., Qwen2.5-32B-Instruct) to perform source-to-source transformations (manual loop unrolling, strength reduction, memory layout changes) on hot functions, then compile the transformed source with Protean's `-OP`. This two-step approach yielded 10.1% speedup on Susan and 8.5% on JPEG in the paper's evaluation.

9. **Validate results.** Always verify that the optimized binary produces correct output on your test suite. Phase ordering can expose latent bugs or change floating-point behavior. Compare output checksums before and after.

10. **Iterate on scope granularity.** If module-level recipes plateau, consider function-level scoping (available in Protean but not yet fully evaluated). This applies different recipes to different functions within the same module, which can help when a module contains both compute-heavy and control-heavy functions.

## Concrete Examples

**Example 1: Optimizing a single-file C benchmark**

User: "I have a compute-heavy image processing program in C. How can I get better performance than -O3?"

Approach:
1. Compile and measure baseline: `clang -O3 -o baseline susan.c -lm && time ./baseline input.pgm`
2. Run Protean with 100 iterations: `clang -OP -mllvm -protean -Wprotean,-max-iterations=100 susan.c -o optimized -lm`
3. Measure optimized: `time ./optimized input.pgm`
4. Protean explores recipes like "A", "CD", "BCE", "ADCE" via simulated annealing
5. Best recipe found: "CD" (GVN/SCCP/BDCE followed by DSE/DCE/inlining)

Output:
```
Baseline (-O3):  0.452s
Protean (-OP):   0.415s  (recipe: "CD", 8.2% speedup)
Build overhead:  12 extra seconds (100 iterations)
```

**Example 2: Extracting IR features for ML research**

User: "I want to build a model that predicts which LLVM passes help a given function. How do I get training features?"

Approach:
1. Compile each benchmark with feature dump enabled:
   ```
   clang -OP -mllvm -protean -Wprotean,-protean-output-table benchmark.c -o /dev/null
   ```
2. Parse the CSV output — each row represents a scope (module, function, loop) with 141 features
3. Pair features with speedup labels by also running the recipe search and recording the best speedup per module
4. Train a model (e.g., Transformer, gradient-boosted trees) mapping features → optimal recipe or predicted speedup

Output (CSV excerpt):
```
scope,name,FunctionCount,AvgBBPerFunc,TotalBBCount,TotalInstCount,...,LoopSize,MaxLoopHeight
module,susan.c,12,8.3,100,1847,...,0,0
function,susan_corners,0,0,22,487,...,0,0
loop,susan_corners.L1,0,0,0,0,...,45,3
```

**Example 3: Two-step LLM + compiler optimization**

User: "I want to combine LLM code optimization with compiler tuning for maximum performance."

Approach:
1. Identify hot functions via profiling (`perf record && perf report`)
2. Feed the hot function source to an LLM with a prompt like: "Optimize this C function for performance. Apply loop unrolling, strength reduction, and memory access pattern improvements. Preserve correctness."
3. Replace the original function with the LLM-optimized version
4. Compile the modified source with Protean: `clang -OP -mllvm -protean -Wprotean,-max-iterations=100 optimized_source.c -o final -lm`
5. Validate correctness, then measure combined speedup

Output:
```
Baseline (-O3 on original):         0.500s
LLM source optimization + -O3:     0.484s  (3.1% speedup)
LLM source optimization + Protean: 0.449s  (10.1% combined speedup)
```

## Best Practices

- **Do:** Start with a small iteration count (10–20) to verify the framework works on your code before scaling to 100+ iterations. The marginal return drops off sharply after ~100 iterations.
- **Do:** Keep recipe length at 5 or below. The paper shows length-5 recipes capture 97% of achievable speedup. Longer recipes add search cost without meaningful gains.
- **Do:** Use IR2Score predictions during search instead of actual execution when your benchmark takes more than a few seconds to run. This converts hours of search time into seconds.
- **Do:** Profile first. Apply Protean to the modules that dominate execution time. Optimizing cold code wastes build time for no measurable benefit.
- **Avoid:** Skipping correctness validation. Different pass orderings can expose undefined behavior or change floating-point results. Always run your test suite on the optimized binary.
- **Avoid:** Assuming one recipe works across all programs. The whole point of phase ordering is that optimal sequences are program-specific. A recipe that speeds up JPEG may slow down a sorting algorithm.
- **Avoid:** Using Protean on debug builds or code compiled with `-O0`/`-O1`. The framework's clusters are derived from `-O3` passes and assume that optimization level as the starting point.

## Error Handling

- **IR unchanged across iterations:** Protean's early-exit detects this automatically. If the IR never changes, the code may already be fully optimized at `-O3`, or the clusters being tested don't apply to this code pattern. Try increasing recipe length or check that the code has optimization opportunities (loops, redundant stores, etc.).
- **Build failures with `-OP`:** Some pass orderings can produce invalid IR if applied to unusual code patterns. Protean should skip failing recipes during search, but if the final recipe causes a crash, fall back to the second-best recipe or reduce recipe length.
- **Feature extraction returns all zeros for loop/callee columns:** This is expected for modules without loops or function calls. The feature set is scope-aware — loop features are only populated for loop-scope rows.
- **Negligible speedup after search:** The program may be memory-bound, I/O-bound, or already near-optimal under `-O3`. Phase ordering primarily helps compute-bound code with optimization-sensitive patterns (loops, inlining opportunities, redundant operations).
- **ML model (IR2Score) predictions diverge from actual speedup:** The model was trained on Polybench/Coral-2 functions. For very different code patterns, retrain on representative samples from your domain.

## Limitations

- Protean is designed for C/C++ compiled with LLVM/Clang. It does not apply to GCC, MSVC, or non-native compilation targets (JVM, .NET, WASM).
- The framework is not yet open-sourced (planned for future release as of the paper's publication). The workflow described here is based on the paper's design and flags. Implementation details may change upon release.
- Module-level granularity may miss optimization opportunities when a single module contains functions with very different optimization profiles. Function-level scoping exists but is not yet thoroughly evaluated.
- The IR2Score model was trained on a limited dataset (10K functions from Polybench and Coral-2). Its predictions may be unreliable on code from very different domains without retraining.
- Build time overhead scales linearly with iteration count and number of modules. For large projects (hundreds of translation units), selective application to hot modules is essential.
- Phase ordering gains are modest on average (4.1%). The technique shines on specific programs where `-O3`'s fixed ordering is a poor fit, not as a universal "make everything faster" tool.

## Reference

[Protean Compiler: An Agile Framework to Drive Fine-grain Phase Ordering](https://arxiv.org/abs/2602.06142v1) — Ashouri et al., 2026. Focus on Section 3 (framework architecture and pass clustering), Section 4 (simulated annealing search), Section 5 (IR2Score model and feature set), and Section 6.2 (LLM two-step optimization results).