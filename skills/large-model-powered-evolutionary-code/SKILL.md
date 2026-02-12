---
name: "large-model-powered-evolutionary-code"
description: "Iteratively optimize code performance using LLM-driven evolutionary search on a phylogenetic tree. Applies PhyloEvolve-style mutation, crossover, elite trajectory pooling, and multi-branch exploration to improve runtime, memory, or correctness of algorithms. Use when: 'optimize this CUDA kernel', 'speed up this numerical solver', 'evolve better implementations of this algorithm', 'iteratively improve this function performance', 'find a faster version of this code', 'reduce memory usage of this computation'."
---

# LLM-Powered Evolutionary Code Optimization (PhyloEvolve)

This skill enables Claude to systematically optimize algorithm implementations through evolutionary search guided by LLM reasoning. Instead of making a single optimization pass, Claude maintains a **phylogenetic tree** of code variants -- tracking parent-child relationships, fitness scores, and mutation history -- then uses that structured history as in-context learning signal to propose increasingly effective modifications. The approach treats sequences of (code change, performance result) pairs as trajectory data, allowing Claude to condition future mutations on what has already worked or failed, without any model fine-tuning.

## When to Use

- When the user asks to **optimize a computationally expensive function** and a single rewrite attempt is unlikely to find the best solution (e.g., GPU kernels, numerical solvers, matrix operations).
- When the user wants to **explore multiple optimization strategies in parallel** rather than committing to one approach.
- When the user says "try several approaches and keep the best one" or "iteratively improve this code."
- When optimizing code where **correctness must be preserved** while improving runtime or memory -- the phylogenetic tree tracks both metrics.
- When the user has a **benchmark or test harness** and wants to use measured performance as the fitness signal driving optimization.
- When a previous optimization attempt regressed performance and the user wants to **backtrack and try a different direction**.

## Key Technique

**PhyloEvolve** reframes code optimization as In-Context Reinforcement Learning (ICRL). The core insight is that every optimization attempt produces a (state, action, reward) tuple -- the current code, the modification applied, and the resulting performance delta. By organizing these tuples into a phylogenetic tree rather than discarding failed attempts, the LLM accumulates a structured optimization history that informs future mutations. High-performing trajectories (sequences of modifications that consistently improved performance) are pooled as "elite trajectories" and injected into prompts to bias future mutations toward productive patterns.

The **phylogenetic tree** is the central data structure. Each node stores: the code variant, its fitness score(s), the mutation that produced it, and its parent lineage. Branches represent independent optimization directions (analogous to "islands" in evolutionary computing). This enables three critical operations: (1) **backtracking** -- when a branch stagnates, revert to an ancestor and try a different mutation; (2) **cross-lineage recombination** -- combine successful patterns from two independent branches; (3) **trajectory replay** -- feed the LLM the full mutation history of the best-performing lineage as context for the next mutation.

The **multi-island strategy** runs several independent optimization lineages in parallel, each pursuing different strategies (e.g., one island focuses on memory layout, another on algorithmic complexity, a third on vectorization). Periodically, the best variant from each island "migrates" to others, introducing diversity and preventing premature convergence. Combined with containerized execution for safe benchmarking, this creates a robust optimization loop.

## Step-by-Step Workflow

1. **Establish the baseline.** Read the target code, identify the function(s) to optimize, and run the existing benchmark or test suite to record baseline metrics (runtime, memory, correctness). Store this as the root node of the phylogenetic tree.

2. **Define the fitness function.** Determine what "better" means for this task: faster wall-clock time, lower peak memory, maintained numerical accuracy, or a weighted combination. Write a scoring function or use the user's existing benchmark. Correctness is always a hard constraint -- variants that break tests are assigned fitness zero.

3. **Analyze the code for optimization opportunities.** Identify 3-5 independent optimization axes (e.g., loop structure, memory access pattern, algorithmic complexity, parallelism, data layout). Each axis will seed a separate "island" (branch in the tree).

4. **Generate the initial population.** For each optimization axis, produce 1-2 variant implementations via targeted mutations. Record each as a child node of the root, annotating the mutation type applied. Run benchmarks and record fitness scores.

5. **Select elites and build trajectory context.** Rank all variants by fitness. Extract the top-performing lineages -- the sequence of mutations from root to each elite node. Format these as trajectory prompts:
   ```
   Trajectory: root -> [mutation: unrolled inner loop] (1.8x speedup)
              -> [mutation: replaced branching with branchless select] (2.1x speedup)
   ```

6. **Propose next-generation mutations.** For each active branch, construct a prompt containing: (a) the current best code on that branch, (b) its fitness score and the delta from its parent, (c) elite trajectories from other branches, and (d) a directive to propose a specific, targeted modification. Generate 1-2 child variants per branch.

7. **Evaluate offspring in isolation.** Run each new variant against the full test suite and benchmark in a clean environment (use a subprocess, container, or tmp directory). Record runtime, memory, and correctness. Discard variants that fail correctness checks.

8. **Update the tree and prune.** Add surviving offspring as new nodes. If a branch has not improved for 2-3 generations, mark it stagnant. Consider backtracking: select an ancestor node from that branch and apply a different mutation strategy. Optionally perform **crossover** -- combine code segments from two high-performing branches into a new variant.

9. **Migrate between islands.** Every 2-3 generations, take the best variant from each island and introduce it as a new branch on other islands. This cross-pollinates successful strategies.

10. **Terminate and report.** Stop when: the fitness improvement plateaus below a threshold, the computational budget is exhausted, or the user's target performance is met. Present the best variant alongside a summary of the phylogenetic tree showing which mutation paths were most productive.

## Concrete Examples

**Example 1: Optimizing a matrix multiplication kernel**

User: "This naive matrix multiply is too slow for my 4096x4096 matrices. Can you iteratively optimize it?"

Approach:
1. Read the baseline implementation. Benchmark it: 12.4 seconds for 4096x4096.
2. Define fitness as inverse wall-clock time with correctness verified against numpy reference.
3. Identify optimization axes: (A) loop tiling for cache, (B) SIMD vectorization, (C) algorithmic change to Strassen-like approach.
4. Generate initial variants:
   - Branch A: Add 64x64 tile blocking -> benchmark: 4.1s (3.0x speedup)
   - Branch B: Use numpy vectorized operations -> benchmark: 0.8s (15.5x speedup)
   - Branch C: Implement recursive Strassen -> benchmark: 5.2s (2.4x speedup)
5. Elite trajectory: Branch B is leading. Feed its mutation history to other branches.
6. Generation 2:
   - Branch A: tile blocking + transpose for stride-1 access -> 2.9s (4.3x)
   - Branch B: vectorized + preallocated output buffer -> 0.6s (20.7x)
   - Branch C: stagnant, backtrack to root, try blocking + Strassen hybrid -> 3.8s
7. Generation 3:
   - Branch B: add parallel chunking with multiprocessing -> 0.19s (65x)
   - Migrate Branch B's best to A: tiling + vectorized inner kernel -> 0.31s
8. Report: Best variant is Branch B gen 3 at 0.19s. Show tree of all 9 variants explored.

Output:
```python
# Best variant (Branch B, Generation 3): 65x speedup over baseline
# Lineage: naive -> numpy vectorized -> preallocated buffers -> parallel chunks
import numpy as np
from multiprocessing import Pool

def matmul_optimized(A, B, n_workers=4):
    n = A.shape[0]
    chunk_size = n // n_workers
    C = np.empty((n, n), dtype=A.dtype)

    def compute_chunk(start):
        end = min(start + chunk_size, n)
        return start, A[start:end] @ B

    with Pool(n_workers) as pool:
        for start, block in pool.map(compute_chunk, range(0, n, chunk_size)):
            C[start:start+block.shape[0]] = block
    return C
```

**Example 2: Reducing memory usage of a graph algorithm**

User: "My PageRank implementation runs out of memory on graphs with 10M+ nodes. Optimize for memory."

Approach:
1. Baseline: dense adjacency matrix uses O(n^2) memory. Fails at 10M nodes.
2. Fitness: peak RSS memory during execution, with correctness measured as L2 norm of rank vector vs. reference < 1e-6.
3. Optimization axes: (A) sparse representation, (B) streaming/out-of-core, (C) in-place updates.
4. Initial variants:
   - Branch A: scipy.sparse CSR matrix -> peak memory 2.1GB for 10M nodes (fits)
   - Branch B: memory-mapped edge list with streaming aggregation -> 800MB
   - Branch C: in-place power iteration on CSR -> 1.8GB
5. Elite: Branch B leads on memory. Feed its trajectory to others.
6. Generation 2:
   - Branch A: CSR + float32 instead of float64 -> 1.1GB
   - Branch B: streaming + quantized ranks (float16 intermediate) -> 420MB
   - Crossover (A+B): CSR with streaming batch updates -> 650MB
7. Report: Branch B gen 2 at 420MB, correctness verified.

Output:
```
Phylogenetic Tree Summary:
root (OOM at 10M)
├── A: sparse CSR [2.1GB] -> A1: float32 CSR [1.1GB]
├── B: streaming [800MB] -> B1: streaming+quantized [420MB] *** BEST ***
├── C: in-place [1.8GB] (stagnant, pruned)
└── AB-cross: CSR+streaming [650MB]

Best: Branch B1 -- streaming edge aggregation with float16 intermediates
Memory reduction: >100x vs baseline (OOM -> 420MB)
Correctness: L2 norm vs reference = 3.2e-7 (within tolerance)
```

**Example 3: Speeding up a PDE solver**

User: "Evolve a faster version of this finite difference heat equation solver."

Approach:
1. Baseline: Python nested loops over 2D grid, 45s for 1000x1000 grid, 100 timesteps.
2. Fitness: wall-clock time. Correctness: max absolute error vs. reference < 1e-8.
3. Axes: (A) numpy vectorization, (B) numba JIT, (C) algorithmic (larger timestep with implicit method).
4. Initial population:
   - A: vectorized stencil with array slicing -> 1.2s (37.5x)
   - B: numba @njit on inner loop -> 0.9s (50x)
   - C: Crank-Nicolson implicit with scipy sparse solve -> 8.1s (5.6x)
5. Generation 2:
   - A: vectorized + pre-computed boundary mask -> 0.95s
   - B: numba + parallel=True + cache=True -> 0.31s (145x)
   - C: stagnant, prune
6. Generation 3:
   - B: numba parallel + loop tiling for L1 cache -> 0.22s (205x)
   - Migrate B to A: vectorized core wrapped in numba -> 0.28s
7. Best: Branch B gen 3 at 0.22s, 205x speedup, max error 4.1e-10.

## Best Practices

**Do:**
- Always verify correctness before considering performance. A fast wrong answer is worthless. Run the full test suite after every mutation.
- Record the exact mutation description on each tree edge (e.g., "replaced nested loop with numpy broadcasting on lines 14-22"). This is the trajectory data that makes future mutations smarter.
- Keep at least 2-3 independent branches alive to maintain diversity. Premature convergence to one strategy misses cross-branch recombination opportunities.
- Use the phylogenetic tree to explain your optimization process to the user. Show which paths were explored, which stagnated, and why the winner won.

**Avoid:**
- Do not apply many mutations at once in a single generation. Make one targeted change per variant so you can attribute performance changes to specific modifications.
- Do not discard failed variants silently. Record them in the tree with their fitness -- negative results inform future mutations by ruling out unproductive directions.
- Do not run benchmarks in a noisy environment. Use process isolation, disable GC during timing, and take median of multiple runs to get stable fitness measurements.
- Do not continue evolving a branch that has stagnated for 3+ generations without backtracking. This wastes budget on a local optimum.

## Error Handling

- **Variant produces runtime error**: Assign fitness zero, record the error message as metadata on the tree node, and use the error as negative signal in future prompts ("Previous attempt to unroll this loop caused IndexError due to boundary conditions").
- **Benchmark results are noisy/inconsistent**: Increase the number of benchmark runs and use median. If variance exceeds 10% of mean, flag the measurement as unreliable and re-run in a more controlled environment.
- **All branches stagnate simultaneously**: This usually means the easy optimizations are exhausted. Try: (a) increasing LLM temperature for more creative mutations, (b) introducing a completely new optimization axis not yet explored, (c) performing cross-lineage recombination between the top 2 branches.
- **Crossover produces invalid code**: This is common because merging code from different branches can create conflicts. Validate syntax before benchmarking. If invalid, retry the crossover with explicit instructions to the LLM about which segments to preserve from each parent.
- **Memory/time budget exhausted**: Return the current best variant with its full lineage trace. The phylogenetic tree is still valuable -- the user can resume optimization later from any node.

## Limitations

- **Requires a runnable benchmark.** The evolutionary loop depends on measurable fitness. If the user has no test suite or timing harness, you must build one first, which adds overhead.
- **Not cost-effective for trivial optimizations.** If a single obvious optimization (e.g., "use numpy instead of a Python loop") gives sufficient speedup, the full evolutionary machinery is overkill. Use this skill when the optimization landscape is genuinely complex.
- **LLM context window bounds the trajectory length.** Very long optimization histories (20+ generations with large code variants) may exceed context limits. Summarize older trajectory entries and keep only the most informative mutations in the prompt.
- **Hardware-specific optimizations may not transfer.** A variant optimized for one GPU/CPU architecture may regress on another. If the user needs portable performance, include multi-hardware fitness evaluation.
- **Crossover is fragile for semantically coupled code.** When two branches have diverged significantly in structure, recombination often produces broken code. Prefer crossover between branches that share structural similarity.

## Reference

**Paper:** "Large Language Model-Powered Evolutionary Code Optimization on a Phylogenetic Tree" (Zhao et al., 2026). arXiv: [2601.14523](https://arxiv.org/abs/2601.14523v1).
**Key insight:** Treating the history of code mutations and their performance outcomes as in-context reinforcement learning trajectories -- organized in a phylogenetic tree -- lets an LLM propose increasingly effective optimizations without any fine-tuning, by conditioning on what has worked (and failed) in prior generations.