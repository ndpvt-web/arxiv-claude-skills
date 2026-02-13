---
name: "magellan-autonomous-discovery-compiler"
description: "Evolve compiler optimization heuristics by coupling LLM code generation with evolutionary search and autotuning. Synthesizes executable C++ decision logic that integrates directly into LLVM or other compilers, replacing hand-crafted rules with empirically validated policies. Use when: 'optimize LLVM inlining heuristic', 'evolve compiler pass logic', 'synthesize register allocation priority', 'automate compiler heuristic design', 'generate C++ optimization policy', 'replace hand-tuned compiler rules with learned ones'."
---

# Magellan: Autonomous Discovery of Compiler Optimization Heuristics

This skill enables Claude to design and implement an agentic framework that evolves compiler optimization heuristics through LLM-driven code synthesis combined with evolutionary search and black-box autotuning. Rather than hand-crafting heuristic rules, Claude guides users through building a closed-loop system where an LLM generates candidate C++ decision logic, a compiler toolchain evaluates candidates against real macro-benchmarks, and evolutionary operators refine the population toward heuristics that match or surpass expert baselines -- as demonstrated for LLVM function inlining (5.23% binary-size reduction, 0.61% performance speedup), register allocation, and XLA graph optimization.

## When to Use

- When the user wants to replace hand-crafted compiler heuristics with automatically discovered policies
- When optimizing LLVM inlining decisions for binary size reduction or end-to-end performance
- When designing a register allocation priority function for `RegAllocGreedy` or similar allocators
- When building an evolutionary code-generation loop that synthesizes C++ compiler logic
- When the user needs to automate heuristic search for any compiler pass (LLVM, GCC, XLA, MLIR)
- When creating a framework that couples LLM code generation with benchmark-driven fitness evaluation
- When exploring how to parameterize generated heuristics with tunable constants and run black-box autotuning (e.g., via Vizier or Optuna)

## Key Technique

**LLM + Evolutionary Search + Autotuning in a Closed Loop.** Magellan's core insight is decomposing heuristic discovery into three cooperating subsystems. An LLM coding agent generates candidate C++ decision functions that conform to the compiler's existing API (e.g., `InlineAdvisor::getAdviceImpl`). These candidates are compiled into the target compiler, evaluated on user-provided macro-benchmarks (measuring metrics like `.text` section size via `llvm-size` or wall-clock time via `perf stat`), and scored. An evolutionary search (AlphaEvolve-style) maintains a population of candidates, applying mutation (LLM-driven code edits within `EVOLVE-BLOCK` delimiters) and crossover to explore the heuristic space across generations.

**Decoupled Structure and Constants.** A critical design choice is having the LLM emit a *policy template* where tunable numerical thresholds are declared as named constants (e.g., `TINY_FUNCTION_THRESHOLD`, `SINGLE_USE_INLINE_BONUS`) exposed as compiler flags. This lets the LLM focus on high-level decision logic while a separate black-box autotuner (e.g., Google Vizier) optimizes the constants. This decoupling drops the invalid-sample rate from over 65% (when the LLM proposes full heuristics end-to-end) to only 13%, yielding a 5x reduction in wasted computation. In practice, 10 outer iterations of autotuning with 10 candidate configurations each achieves strong results in roughly 5 hours.

**Compact, Integrable Output.** The evolved heuristics are production-ready: the best inlining policy is 143 lines of executable C++ logic -- 15x shorter than the 2,115-line manual implementation it replaces. It uses weighted instruction counting with category-based complexity weights, conditional branching on function attributes (AlwaysInline, NoInline, OptimizeForSize, Hot), and type-matching analysis, all expressed through standard LLVM APIs.

## Step-by-Step Workflow

1. **Identify the target heuristic and its API contract.** Locate the compiler decision function to replace (e.g., `InlineAdvisor::getAdviceImpl(CallBase &CB)` for inlining, or the priority function in `RegAllocGreedy` for register allocation). Document the function signature, return type, and available input features (callee instruction count, basic block count, function attributes, caller context).

2. **Define the evaluation harness and fitness metric.** Select macro-benchmarks representative of production workloads (e.g., compiling clang itself with PGO + ThinLTO at `-O3`). Define the fitness function: `.text` section size for binary-size optimization, wall-clock time via `perf stat` for performance, or a weighted combination. Use the final linked binary for measurement, not intermediate object files.

3. **Scaffold the policy template with EVOLVE-BLOCK markers.** Write a skeleton C++ file that conforms to the compiler's API. Mark the editable region with `// EVOLVE-BLOCK-START` and `// EVOLVE-BLOCK-END` delimiters. Declare all tunable numerical thresholds as named constants at the top of the file, exposed as compiler flags (e.g., `-mllvm -inline-tiny-threshold=50`). This separates structure from constants.

4. **Configure the LLM code-generation prompt.** Provide the LLM with: (a) the API documentation for available features (38 predefined features for the feature-based variant, or full LLVM API access for the API-based variant), (b) the skeleton template with EVOLVE-BLOCK markers, (c) the optimization objective in natural language, and (d) examples of valid decision logic. Instruct the LLM to emit compilable C++ that respects the function signature.

5. **Implement the evolutionary search loop.** Initialize a population of candidate heuristics by sampling the LLM multiple times. For each generation: (a) evaluate all candidates by compiling them into the compiler and running benchmarks, (b) rank by fitness score, (c) select parents via tournament or fitness-proportional selection, (d) produce offspring through LLM-driven mutation (prompt the LLM to modify the EVOLVE-BLOCK region of a parent) and crossover (combine logic from two parents).

6. **Run black-box autotuning on the best template.** After evolutionary search converges on a structural template, freeze the code logic and optimize the named constants using a black-box optimizer (Vizier, Optuna, or similar). Propose batches of 10 hyperparameter configurations per iteration; run 10 outer iterations for size tasks or use batches of 40 for performance tasks where measurement noise is higher.

7. **Validate correctness and generalization.** Verify the evolved heuristic produces correct compiler output on a comprehensive test suite (e.g., LLVM's `check-all`). Test temporal generalization by evaluating on compiler snapshots from different dates. Test domain generalization by running on diverse production binaries outside the training benchmark.

8. **Integrate and measure against baselines.** Replace the original heuristic in the compiler source tree with the evolved policy. Benchmark against: (a) the original hand-crafted heuristic, (b) LLVM's `-Oz` for size or `-O3` for performance, and (c) any ML-based baselines. Report metrics on both training and held-out benchmarks.

9. **Iterate on the search space if needed.** If the evolved heuristic plateaus, expand the feature set (e.g., from 38 predefined features to full API access), seed the population with the best-so-far policy rather than starting from scratch, or refine the fitness function (e.g., switch from pure size to a Pareto objective balancing size and performance).

## Concrete Examples

**Example 1: Evolving an LLVM Inlining Heuristic for Binary Size**

User: "I want to automatically discover an inlining heuristic that minimizes binary size for our production binary."

Approach:
1. Identify target: `AEInlineAdvisor::getAdviceImpl(CallBase &CB)` in LLVM's inlining pass
2. Define fitness: `.text` section size of the linked production binary, measured via `llvm-size`
3. Scaffold the template:

```cpp
// EVOLVE-BLOCK-START
// Tunable constants (optimized by autotuner)
static constexpr int TINY_FUNCTION_THRESHOLD = 50;    // -mllvm flag
static constexpr int SINGLE_USE_INLINE_BONUS = 80;
static constexpr double COMPLEXITY_WEIGHT_CALL = 1.5;
static constexpr double COMPLEXITY_WEIGHT_BRANCH = 1.2;

std::unique_ptr<InlineAdvice> AEInlineAdvisor::getAdviceImpl(CallBase &CB) {
  Function *Callee = CB.getCalledFunction();
  if (!Callee || Callee->isDeclaration())
    return getDefaultAdvice(CB);

  // Always respect explicit attributes
  if (Callee->hasFnAttribute(Attribute::AlwaysInline))
    return getAdvice(CB, /*ShouldInline=*/true);
  if (Callee->hasFnAttribute(Attribute::NoInline))
    return getAdvice(CB, /*ShouldInline=*/false);

  int InstCount = Callee->getInstructionCount();
  // Weighted cost model with category-based complexity
  double Cost = computeWeightedCost(Callee, COMPLEXITY_WEIGHT_CALL,
                                     COMPLEXITY_WEIGHT_BRANCH);
  bool IsTiny = (InstCount < TINY_FUNCTION_THRESHOLD);
  bool IsSingleUse = Callee->hasOneUse();

  int Bonus = IsSingleUse ? SINGLE_USE_INLINE_BONUS : 0;
  bool ShouldInline = (Cost - Bonus < getThreshold());
  return getAdvice(CB, ShouldInline);
}
// EVOLVE-BLOCK-END
```

4. Run evolutionary search for ~50 generations, evaluating each candidate by compiling clang with the heuristic and measuring `.text` size
5. Autotuning: freeze best template, optimize constants over 10 iterations x 10 configs
6. Expected outcome: 4-6% binary size reduction vs. default LLVM heuristic; 143 lines vs. 2,115 lines in the original

**Example 2: Register Allocation Priority Rule**

User: "Can we learn a better priority function for LLVM's greedy register allocator?"

Approach:
1. Target the priority-queue policy in `RegAllocGreedy` that determines live-range processing order
2. Scaffold a priority function that takes live-range features (spill weight, size, number of uses, interference count) and returns a priority score
3. Run evolutionary search with fitness = runtime of a large-scale workload (e.g., SPEC CPU)

```cpp
// EVOLVE-BLOCK-START
float computeLiveRangePriority(const LiveInterval &LI,
                                const MachineRegisterInfo &MRI) {
  // Evolved priority: features available include
  // LI.weight(), LI.getSize(), number of uses, register class pressure
  float SpillWeight = LI.weight();
  unsigned Size = LI.getSize();
  // LLM discovers the priority logic here
  return SpillWeight;  // Initial seed -- evolved by search
}
// EVOLVE-BLOCK-END
```

4. Key finding from the paper: evolutionary search may converge to a constant-value policy, revealing that insertion order alone (not a complex priority function) suffices for strong performance. This is a genuine discovery -- the framework can surface non-obvious simplifications.

**Example 3: XLA Graph Rewriting Heuristic**

User: "We want to optimize extraction policies for equality saturation in our XLA-based compiler."

Approach:
1. Formulate as selecting an optimized graph from an e-graph produced by equality saturation
2. Define fitness: end-to-end execution time of the extracted computation graph
3. The LLM generates a cost-model function that scores candidate extractions
4. Evolutionary search + autotuning discovers policies achieving ~7% improvement over manual baselines
5. This demonstrates the framework's portability beyond LLVM with minimal engineering overhead

## Best Practices

- **Do:** Decouple decision logic from numerical constants. Have the LLM generate the code structure while a black-box autotuner optimizes thresholds. This reduces invalid samples from 65% to 13%.
- **Do:** Use macro-benchmarks (full application builds) rather than microbenchmarks for fitness evaluation. Measure the final linked binary, not intermediate compilation units.
- **Do:** Seed performance-oriented search with a policy already optimized for size. Starting from scratch for performance tasks often plateaus at regressions; seeding with a good size policy provides a better starting point.
- **Do:** Mark editable regions with explicit `EVOLVE-BLOCK` delimiters so the LLM modifies only the heuristic logic, not surrounding compiler infrastructure.
- **Avoid:** Letting the LLM jointly propose both code structure and numerical constants in a single shot. This produces a 65%+ rate of non-compiling or degenerate candidates.
- **Avoid:** Evaluating on random function samples rather than full application builds. Per-function metrics don't capture cross-function interactions (e.g., inlining decisions affect downstream optimizations).

## Error Handling

- **Non-compiling candidates:** Wrap each candidate evaluation in a compilation check. If the modified compiler fails to build, assign the worst possible fitness score and discard. Track the invalid rate per generation -- if it exceeds 30%, simplify the EVOLVE-BLOCK interface or add more constraints to the LLM prompt.
- **Correctness regressions:** Run the compiler's test suite (`check-all` for LLVM) after each candidate. Any test failure means the candidate is invalid regardless of its fitness score. Compiler heuristics affect only optimization quality, not correctness -- but bugs in generated code can violate this invariant.
- **Fitness noise in performance measurements:** Use multiple benchmark runs (3-5 repetitions) and take the median. For performance-sensitive tasks, use larger autotuning batches (40 configs instead of 10) to compensate for measurement variance.
- **Search stagnation:** If fitness plateaus for 10+ generations, inject diversity by re-sampling fresh candidates from the LLM, expanding the feature set (e.g., from 38 predefined features to full API access), or increasing the mutation rate.

## Limitations

- **Requires a working compiler build system.** Each candidate evaluation involves a full compiler rebuild and benchmark run. This is computationally expensive (the paper reports 1.5 days for sequential exploration, ~5 hours with autotuning for 10 iterations). Infrastructure for distributed builds significantly accelerates the loop.
- **Macro-benchmark dependency.** The evolved heuristic is only as good as the benchmark it optimizes against. Overfitting to a single benchmark is a real risk -- always validate on held-out workloads and test temporal/domain generalization.
- **LLVM API surface complexity.** The full API-based variant (giving the LLM access to arbitrary LLVM APIs) produces better results than the feature-based variant but is harder to prompt correctly. Start with the constrained feature-based approach before graduating to full API access.
- **Not a replacement for correctness-critical logic.** This approach works for optimization heuristics where any valid decision is correct (just possibly suboptimal). It is not suitable for compiler passes where incorrect decisions produce wrong code.
- **LLM-dependent.** The quality of evolved heuristics depends on the LLM's ability to generate valid C++ that uses compiler APIs correctly. Stronger code-generation models produce better initial populations and more productive mutations.

## Reference

[Magellan: Autonomous Discovery of Novel Compiler Optimization Heuristics with AlphaEvolve](https://arxiv.org/abs/2601.21096v1) (Chen et al., 2026). Focus on Section 3 (framework architecture), Section 4.1 (inlining heuristic synthesis with EVOLVE-BLOCK structure and autotuning decoupling), and Section 4.3 (the surprising constant-priority finding in register allocation). The key quantitative takeaway: 143 lines of evolved C++ outperform 2,115 lines of hand-crafted LLVM inlining logic.