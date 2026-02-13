---
name: "colt-lightweight-multi-llm-collaboration"
description: |
  Orchestrate multiple LLMs of different sizes through a shared MCTS tree to solve
  compiler optimization, code transformation sequencing, and multi-step search problems
  at lower cost than a single large model. Uses model-aware tree policies, joint actions
  (task step + next model), and automatic escalation when smaller models regress.

  Trigger phrases:
  - "Optimize this compiler pass ordering using multiple models"
  - "Use MCTS with mixed model sizes to find the best transformation sequence"
  - "Set up a lightweight multi-LLM search for compiler optimization"
  - "Route simple optimization decisions to small models, hard ones to large"
  - "Find the best TVM/XLA schedule using collaborative tree search"
  - "Reduce LLM serving cost for iterative code optimization"
---

# COLT: Lightweight Multi-LLM Collaboration through Shared MCTS Reasoning

This skill enables Claude to design and implement multi-LLM collaborative search systems based on the COLT framework. Instead of routing every compiler optimization or code transformation decision through a single expensive large model, COLT embeds model selection directly inside a Monte Carlo Tree Search loop. A shared MCTS tree lets cheap small models handle routine expansion steps while a larger model is called only when the search detects persistent regression. The result is near-large-model quality at a fraction of the inference cost -- applicable to compiler pass ordering, build configuration search, LLM-guided code refactoring pipelines, and any sequential decision problem where multiple models can share a search tree.

## When to Use

- When the user needs to **optimize compiler pass orderings** (TVM schedules, XLA HLO passes, LLVM phase ordering) and wants LLM guidance without paying for a large model on every step.
- When building an **LLM-guided search system** where decisions vary in difficulty -- some nodes are trivial (small model) and some require deep reasoning (large model).
- When the user asks to **reduce inference cost** of an existing single-LLM optimization loop by introducing a tiered model strategy.
- When implementing **MCTS for code transformations** (loop tiling, operator fusion, memory layout selection) and wanting multiple models to contribute to the same search tree.
- When designing a **multi-model orchestration pipeline** that avoids heavy agentic scaffolding (no external planner, no database, no controller) by internalizing routing in the tree policy.
- When the user wants to **benchmark small-vs-large model collaboration** on sequential optimization tasks.

## Key Technique

COLT's core insight is that a **single shared MCTS tree** is sufficient as the collaboration substrate between models of different sizes. Traditional multi-agent setups require external planners, message buses, and state databases. COLT avoids all of that: the tree itself stores the shared state. Each node records the compiler transformation applied, the model that proposed it, and backpropagated value statistics. When any model expands a node, the result is visible to every subsequent model that traverses that path. This means a small model's successful prefix can be extended by a large model (or vice versa), enabling **cross-model value propagation** without explicit communication.

The **joint action space** is the key mechanism. At each MCTS iteration, the acting model proposes not just the next compiler transformation (e.g., "tile loop by 4", "fuse operators A and B") but also **which model should be queried next**. This turns model routing into a learned part of the search rather than an external policy. A model-aware UCB tree policy biases selection toward smaller models by incorporating a cost penalty term, so the search naturally prefers cheap expansions unless the value estimates justify escalation.

The **course-alteration mechanism** monitors for persistent regression: if the last K expansions by small models all decreased the best-known value, COLT escalates to the largest available model for the next N steps. This is a simple sliding-window heuristic, not a learned policy, making it easy to implement and tune. After the large model stabilizes the search, control returns to the small model pool.

## Step-by-Step Workflow

1. **Define the transformation vocabulary.** Enumerate the set of compiler passes or code transformations that form the action space. For TVM, this might be `[tile, unroll, vectorize, fuse, reorder, parallelize, cache_read, cache_write]` with associated parameters. Store these as a structured list with costs and preconditions.

2. **Configure the model tier list.** Assign each available LLM a tier with estimated cost-per-token and capability rating. Example: `{tier_0: "claude-haiku", tier_1: "claude-sonnet", tier_2: "claude-opus"}`. Set the default starting tier to the smallest model.

3. **Initialize the shared MCTS tree.** Create a root node representing the unoptimized program. Each node stores: `(transformation_applied, model_that_proposed, visit_count, cumulative_value, children)`. The value is the measured speedup (or inverse runtime) of the program after applying the transformation sequence from root to that node.

4. **Implement the model-aware UCB selection policy.** At each node, compute the selection score as:
   ```
   score(child) = Q(child)/N(child) + c * sqrt(ln(N(parent))/N(child)) - lambda * cost(child.model)
   ```
   where `lambda` is a cost-sensitivity hyperparameter and `cost(child.model)` is the normalized per-token cost of the model that would be queried for expansion. This biases selection toward paths expanded by cheaper models.

5. **Run the MCTS loop with joint actions.** For each iteration: (a) **Select** a leaf using the model-aware UCB policy. (b) **Expand** by prompting the selected model with the current transformation prefix and asking it to propose `(next_transformation, next_model_tier)` as a joint action. (c) **Simulate** by compiling and benchmarking the program with the new transformation appended. (d) **Backpropagate** the measured speedup through all ancestor nodes.

6. **Prompt the LLM with structured context.** Each model call includes: the original program IR, the transformation sequence so far, the best speedup found, and the last 3 backpropagated values. Ask the model to output a JSON object: `{"transformation": "tile_loop_i_by_8", "next_model": "tier_0"}`. Small models tend to suggest staying at their tier; the large model may delegate back down.

7. **Implement course-alteration detection.** Track a sliding window of the last `K=5` expansions. If all K decreased the best value and were performed by tier-0 or tier-1 models, force the next `N=3` expansions to use the largest model (tier-2). Reset the window after escalation completes.

8. **Extract the best transformation sequence.** After the MCTS budget is exhausted (e.g., 200 iterations), trace the path from root to the highest-value leaf. This sequence is the recommended compiler pass ordering.

9. **Log model utilization and cost.** Record how many iterations each model tier handled, total tokens consumed, and the final speedup. Compare against a baseline of running all iterations with the largest model to quantify cost savings.

10. **Optionally warm-start future searches.** Persist subtrees that achieved high values. When optimizing a similar program, graft these subtrees onto the new root to accelerate convergence (the shared tree structure makes this straightforward).

## Concrete Examples

**Example 1: TVM Schedule Optimization**

```
User: I have a matrix multiply kernel in TVM. Find the best schedule
      using multiple model tiers to keep cost low.

Approach:
1. Parse the TVM compute definition to extract loop structure
   (batch, M, N, K dimensions).
2. Define action space: [tile_M, tile_N, tile_K, unroll, vectorize,
   bind_gpu, cache_read, cache_write] with parameter ranges.
3. Configure tiers: haiku (tier-0, $0.25/M tokens), sonnet (tier-1,
   $3/M tokens), opus (tier-2, $15/M tokens).
4. Initialize MCTS root with the naive schedule (no transformations).
5. Run 150 iterations. Prompt tier-0 first:

   Prompt to tier-0:
   "Program: matmul(M=1024, N=1024, K=1024)
    Current schedule: [tile_M_by_32, tile_N_by_32]
    Best speedup so far: 3.2x
    Last 3 values: [3.2, 2.8, 3.1]
    Propose next: {transformation, next_model}"

   Tier-0 response:
   {"transformation": "vectorize_inner_by_4", "next_model": "tier_0"}

6. After iteration 80, detect 5 consecutive regressions from tier-0.
   Escalate to tier-2 for 3 iterations. Tier-2 proposes cache_read
   for the shared K dimension -- a non-obvious optimization that
   restores speedup trajectory.
7. Final sequence: [tile_M_32, tile_N_32, vectorize_4, cache_read_K,
   unroll_inner_2, bind_blockIdx]. Speedup: 11.4x.

Output:
  Best schedule: 11.4x speedup
  Model usage: tier-0: 128 iters (85%), tier-1: 12 iters (8%),
               tier-2: 10 iters (7%)
  Estimated cost: $0.42 (vs $3.80 all-opus baseline, 89% savings)
```

**Example 2: LLVM Pass Ordering Search**

```
User: Find the best LLVM -O2 pass ordering for my DSP loop kernel.
      I have access to GPT-4o-mini, GPT-4o, and o3.

Approach:
1. Extract the LLVM IR for the DSP kernel.
2. Define action space: the 67 standard -O2 passes (mem2reg, instcombine,
   loop-rotate, slp-vectorizer, etc.).
3. Tiers: gpt-4o-mini (tier-0), gpt-4o (tier-1), o3 (tier-2).
4. Set MCTS budget to 200 iterations, lambda=0.3 for cost sensitivity.
5. Run the loop. Tier-0 handles straightforward passes (mem2reg early,
   instcombine after simplifications). When the search plateaus at
   1.6x speedup, course-alteration kicks in.
6. Tier-2 identifies that running loop-unswitch BEFORE licm (contrary
   to the default ordering) enables better hoisting for this specific
   kernel's branch pattern.
7. Extract optimal 12-pass sequence. Compile and verify.

Output:
  Optimal pass sequence (12 passes):
    mem2reg -> simplifycfg -> instcombine -> loop-rotate ->
    loop-unswitch -> licm -> indvars -> loop-idiom ->
    slp-vectorizer -> instcombine -> dse -> simplifycfg
  Speedup: 2.1x over default -O2
  Model usage: mini: 160 (80%), 4o: 25 (12.5%), o3: 15 (7.5%)
```

**Example 3: Build Configuration Optimization**

```
User: Use COLT-style search to find optimal CMake build flags for
      my C++ project targeting ARM Cortex-A72. Minimize binary size
      while keeping performance within 5% of -O2.

Approach:
1. Define action space: compiler flags as transformations
   [-Os, -Oz, -flto, -ffunction-sections, -fdata-sections,
    -fno-exceptions, -fno-rtti, -mfpu=neon, -mcpu=cortex-a72, ...].
2. Value function: binary_size_reduction * (1 if perf >= 0.95*O2 else 0).
3. Use haiku for flag compatibility checks (fast, cheap).
   Use sonnet when flag interactions cause unexpected size increases.
   Use opus for architectural trade-off reasoning (LTO + section GC).
4. Run 100 MCTS iterations. Best path: [-Os, -flto, -ffunction-sections,
   -fdata-sections, -Wl,--gc-sections, -fno-rtti, -mfpu=neon].

Output:
  Binary size: 142KB (vs 284KB at -O2, 50% reduction)
  Performance: 97.3% of -O2 (within 5% threshold)
  Cost: $0.18 total inference
```

## Best Practices

- **Do:** Start every search with the smallest model tier. Let the course-alteration mechanism handle escalation naturally rather than pre-assigning tiers to specific phases.
- **Do:** Include the transformation prefix and recent value trajectory in every prompt. Models need context about what has been tried and whether the search is improving.
- **Do:** Set `lambda` (cost penalty) empirically. Start with `lambda=0.1` and increase until the large model usage drops below 15% of total iterations without degrading final solution quality.
- **Do:** Cache compilation/benchmark results keyed by transformation sequence. MCTS may revisit the same prefix through different tree paths -- avoid redundant builds.
- **Avoid:** Letting the large model decide next-model routing exclusively. The joint action should be a suggestion; the tree policy has final say via the UCB cost term.
- **Avoid:** Setting the course-alteration window K too small (< 3). Stochastic variation in benchmarks can trigger false escalations. Use K=5 as a default.
- **Avoid:** Using this approach for single-shot decisions. COLT's value comes from iterative search where the tree accumulates signal over many expansions. For one-off choices, just use the best model directly.

## Error Handling

| Problem | Symptom | Resolution |
|---------|---------|------------|
| Small model proposes invalid transformation | Compilation fails during simulation | Assign value=-1 to the node, backpropagate the penalty, the tree will avoid that subtree. Do not retry with a larger model for the same node. |
| Course-alteration loops indefinitely | Large model also causes regressions | Cap escalation to 2 consecutive invocations. If the large model cannot improve, the search may have converged -- terminate early and return the best-known path. |
| Model returns malformed JSON | Cannot parse joint action | Retry once with a stricter prompt. If it fails again, use a fallback: pick a random valid transformation and stay at the current tier. |
| Benchmark variance masks real improvements | Backpropagated values are noisy | Run each benchmark 3 times and use the median. Alternatively, use a moving average for value updates instead of raw measurements. |
| Tree grows too large for memory | >10K nodes after many iterations | Prune subtrees whose best descendant value is below 50% of the global best. These paths are unlikely to yield improvements. |

## Limitations

- **Requires a measurable objective.** COLT needs a scalar value (speedup, binary size, latency) to backpropagate. Subjective code quality improvements cannot be optimized this way.
- **Compilation/benchmark overhead.** Each MCTS simulation requires actually compiling and running the program. For large projects with long build times, the wall-clock cost may dominate LLM inference cost.
- **Action space must be predefined.** The set of valid transformations needs to be enumerated upfront. Open-ended code rewrites do not fit the discrete action model without additional abstraction.
- **Not suitable for latency-sensitive applications.** MCTS is inherently iterative. If you need a transformation sequence in under a second, use a single model call instead.
- **Model capability gaps matter.** If the small model is too weak to propose any useful transformations, the tree fills with low-value nodes and the large model spends its budget recovering rather than exploring. Ensure tier-0 can handle at least the routine decisions in your domain.

## Reference

[COLT: Lightweight Multi-LLM Collaboration through Shared MCTS Reasoning for Model Compilation](https://arxiv.org/abs/2602.01935v1) -- Tang et al., 2026. Focus on Section 3 (shared MCTS formulation and model-aware UCB) and Section 4 (course-alteration mechanism and experimental cost breakdowns).