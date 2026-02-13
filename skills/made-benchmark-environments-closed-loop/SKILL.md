---
name: "made-benchmark-environments-closed-loop"
description: "Build closed-loop discovery benchmarks where an agent iteratively proposes, evaluates, and refines candidates under a fixed oracle budget. Use when: 'build a materials discovery benchmark', 'create a closed-loop optimization pipeline', 'benchmark an iterative search agent', 'set up a discovery loop with budget constraints', 'evaluate a generative pipeline with oracle feedback', 'compare discovery agents on convex hull stability'."
---

# Closed-Loop Discovery Benchmark Environments (MADE)

This skill enables Claude to build **closed-loop discovery benchmark frameworks** modeled on the MADE architecture (MAterials Discovery Environments). The core idea: instead of evaluating generative or predictive models as isolated components, you wire them into a full propose-evaluate-refine loop under a strict oracle budget, then measure how efficiently the system discovers valid candidates. This pattern applies to any domain where you iteratively search a combinatorial space with expensive evaluations — materials science, drug discovery, molecular design, hyperparameter search, or experimental design.

## When to Use

- When the user wants to benchmark an **iterative search or discovery pipeline** end-to-end, not just individual model accuracy
- When building a system where an agent **proposes candidates, receives oracle feedback, and adapts** its strategy over multiple rounds
- When the user needs to compare **modular pipeline configurations** (swap generators, filters, rankers) under controlled conditions
- When designing an optimization loop with a **fixed evaluation budget** that models real-world cost constraints (lab experiments, API calls, compute)
- When evaluating candidate quality against a **Pareto frontier or convex hull** rather than a single scalar target
- When the user asks to implement a **ReAct-style agentic orchestrator** that dynamically selects which tool to call next based on accumulated results

## Key Technique

MADE formalizes discovery as sequential decision-making under budget. At each iteration `t <= B`, a policy `pi` selects a candidate `s_t` based on the history of all previous `(candidate, evaluation)` pairs. An oracle `O` returns the ground-truth quality `E_t = O(s_t)`, and the candidate joins the known set. The objective is to maximize the number of **stable** (valid) discoveries after `B` total queries. Stability is defined relative to a convex hull: a candidate is stable if its energy above the hull `delta_hull(s, H_t) <= epsilon` (default threshold: 0.1 eV/atom in the materials domain, but generalizable to any frontier-based validity criterion).

The key architectural insight is **modular composition**. A discovery agent is assembled from four interchangeable component types: (1) **Planners** that select which region of the search space to explore (random, diversity-maximizing, or LLM-guided); (2) **Generators** that produce candidate structures given a target specification; (3) **Filters** that remove invalid or redundant candidates cheaply before oracle evaluation; (4) **Selectors** that rank remaining candidates using surrogate models or heuristics. By swapping components while holding others fixed, you can ablate contributions and identify bottlenecks.

Evaluation uses four metrics that separate **what** was discovered from **how fast**: `mSUN` (fraction of stable, unique, novel discoveries), `AUDC` (area under the discovery curve, normalized to [0,1]), `AF` (Acceleration Factor — how many fewer queries this method needs vs. baseline to reach k discoveries), and `EF` (Enhancement Factor — multiplicative improvement in discoveries at query count t). These paired absolute/relative metrics prevent gaming by either lucky early hits or slow-but-thorough strategies.

## Step-by-Step Workflow

1. **Define the search space and oracle.** Specify the candidate domain (e.g., chemical compositions, molecular graphs, design parameters), the oracle function that evaluates a candidate and returns a scalar quality score, and the budget `B` (total allowed oracle calls). Wrap the oracle behind an interface that tracks call count and enforces the budget.

2. **Implement the convex hull (frontier) evaluator.** Build a function that, given all evaluated candidates so far, computes the Pareto frontier or convex hull of the known-stable set. For each new candidate, calculate its distance to the hull (`delta_hull`). Classify candidates as stable/valid when `delta_hull <= epsilon`. Use `scipy.spatial.ConvexHull` or equivalent for numeric domains.

3. **Build the component interfaces.** Define abstract base classes or protocols for `Planner`, `Generator`, `Filter`, and `Selector`. Each takes the current history `H_t = {(s_i, E_i)}` and returns its output. This enforces swappability.

4. **Implement at least one concrete variant per component.** Start with baselines: a random planner, a random structure generator, a validity filter (reject obviously invalid candidates), and a random selector. Then add stronger variants: diversity-maximizing planner, learned generator, surrogate-model selector (e.g., a lightweight ML ranker).

5. **Wire the closed loop.** Implement the main episode loop: for `t` in `1..B`, call `planner.select_target(H_t)`, then `generator.generate(target)`, then `filter.apply(candidates)`, then `selector.rank(candidates, H_t)`, pick the top candidate, call `oracle.evaluate(candidate)`, update `H_t`, and recompute the hull.

6. **Add the agentic orchestrator (optional).** Implement a ReAct-style controller that, instead of following the fixed planner-generator-filter-selector pipeline, uses an LLM to decide which component to invoke next based on the current buffer state. The LLM sees the history and available tools, and outputs a tool call. This enables adaptive strategies like "generate more diverse candidates when recent discoveries stall."

7. **Instrument metrics collection.** After each oracle call, record: cumulative stable discoveries `D(t)`, the discovery curve, and per-candidate `delta_hull`. At episode end, compute `mSUN`, `AUDC`, `AF`, and `EF` relative to the random baseline.

8. **Run ablation experiments.** Define a grid of component configurations (e.g., 3 planners x 2 generators x 2 selectors). Run each configuration for `N` episodes with different random seeds. Store results in a structured format (JSON lines or DataFrame).

9. **Visualize and compare.** Plot discovery curves (cumulative discoveries vs. oracle queries) for all configurations. Compute and tabulate `AF` and `EF` with confidence intervals. Highlight which component swap produces the largest acceleration.

10. **Package as a reproducible benchmark.** Use config files (YAML or dataclass-based) to specify each pipeline variant. Provide a CLI entry point: `python run_benchmark.py --config configs/chemeleon_mlip.yaml --budget 200 --seed 42`.

## Concrete Examples

**Example 1: Benchmarking molecular generation strategies**

User: "I have a molecular generator and an expensive DFT oracle. I want to compare random generation vs. my trained model in a closed-loop discovery setting with a budget of 100 evaluations."

Approach:
1. Define the oracle interface wrapping the DFT calculator with a call counter and budget=100.
2. Implement `ConvexHullEvaluator` using formation energies across compositions.
3. Create two generator variants: `RandomGenerator` and `TrainedModelGenerator`.
4. Use identical planner (random composition selection), filter (SMACT validity), and selector (random) for both.
5. Run 20 episodes per configuration, collect discovery curves.
6. Compute AF: if trained model reaches 10 stable discoveries in 40 queries vs. 100 for random, AF=2.5.

Output:
```python
# results summary
Config              | mSUN  | AUDC  | AF@10 | EF@50
--------------------|-------|-------|-------|------
random_generator    | 0.12  | 0.31  | 1.0   | 1.0
trained_generator   | 0.38  | 0.67  | 2.5   | 3.2
```

**Example 2: Adding a surrogate selector to an existing pipeline**

User: "My discovery pipeline already uses a learned generator. I want to see if adding an MLIP-based ranker before oracle evaluation improves efficiency."

Approach:
1. Keep planner, generator, and filter fixed across both configurations.
2. Implement `MLIPSelector` that scores each candidate with a pre-trained interatomic potential and passes only the top-k to the oracle.
3. Implement `RandomSelector` as baseline (pass candidates in random order).
4. Run both configurations with budget=200 over 30 seeds.
5. Plot discovery curves side by side and compute AF.

Output:
```
Discovery curve comparison (averaged over 30 seeds):
- Without MLIP selector: 8.2 stable discoveries at B=200
- With MLIP selector:    22.7 stable discoveries at B=200
- Enhancement Factor at B=200: EF = 2.77
- Acceleration Factor to reach 8 discoveries: AF = 4.1
```

**Example 3: Building a ReAct-style agentic discovery loop**

User: "I want to build an LLM-based agent that can dynamically choose between exploring new compositions and exploiting promising ones during a discovery campaign."

Approach:
1. Define tools: `explore_new_composition()`, `generate_variants(composition)`, `score_with_surrogate(candidates)`, `submit_to_oracle(candidate)`.
2. Build a ReAct controller: at each step, the LLM sees the current discovery history, remaining budget, and available tools, then emits a tool call.
3. Implement the system prompt: "You are a materials discovery agent. You have {remaining_budget} oracle queries left. You have discovered {n_stable} stable materials. Decide what to do next."
4. Run alongside the best fixed pipeline as comparison.
5. Evaluate whether the agent's adaptive strategy (e.g., shifting from exploration to exploitation as budget depletes) outperforms the static pipeline.

Output:
```
Agent strategy trace (first 10 steps):
t=1:  explore_new_composition -> Ti-Al-Ni
t=2:  generate_variants(Ti-Al-Ni) -> 50 candidates
t=3:  score_with_surrogate(candidates) -> top-5 ranked
t=4:  submit_to_oracle(TiAl2Ni) -> delta_hull=0.03, STABLE
t=5:  generate_variants(Ti-Al-Ni) -> 50 more variants  [exploiting success]
t=6:  submit_to_oracle(Ti3AlNi2) -> delta_hull=0.08, STABLE
t=7:  submit_to_oracle(TiAlNi3) -> delta_hull=0.42, unstable
t=8:  explore_new_composition -> Fe-Co-Mn  [switching to exploration]
...
Final: Agent AF=5.4 vs best fixed pipeline AF=6.4
```

## Best Practices

- **Do** enforce the oracle budget strictly at the interface level. Never allow components to "peek" at oracle results without decrementing the counter. Budget leaks invalidate all comparisons.
- **Do** seed everything (random generators, model sampling, planner choices) and run multiple episodes. Discovery is stochastic; report means with confidence intervals, not single runs.
- **Do** always include a random baseline. The Acceleration Factor is meaningless without a well-characterized reference. Random search is the universal denominator.
- **Do** recompute the convex hull after every oracle evaluation, not in batches. The hull changes as new points arrive, and stability classification of earlier candidates can shift.
- **Avoid** evaluating generators or filters in isolation. The whole point of closed-loop benchmarking is measuring how components interact. A filter that removes 90% of candidates is useless if the remaining 10% are all unstable.
- **Avoid** conflating exploration budget with wall-clock time. The oracle call count is the primary cost metric. Internal compute (generation, filtering, surrogate scoring) is considered free relative to oracle cost.

## Error Handling

- **Oracle returns NaN or error**: Mark the candidate as "failed evaluation," still decrement the budget counter (failed experiments consume real resources), and log the failure. Do not retry automatically — this would distort budget accounting.
- **Convex hull computation fails** (e.g., fewer than d+1 points in d dimensions): Fall back to per-element comparison or skip hull-based stability until enough data accumulates. Track how many iterations run in "pre-hull" mode.
- **Generator produces no valid candidates after filtering**: Log the event, skip to next iteration with budget unchanged (no oracle call was made). If this happens repeatedly, the planner-generator-filter combination is incompatible — surface this as a diagnostic.
- **Surrogate selector disagrees strongly with oracle**: Track rank correlation between surrogate scores and oracle values over time. If correlation drops below a threshold, consider retraining the surrogate or falling back to random selection.

## Limitations

- The framework assumes oracle evaluations are the **dominant cost**. If generation or surrogate scoring is also expensive, the budget model needs extension to account for multi-resource costs.
- Convex hull stability is specific to thermodynamic phase diagrams. For other domains (drug discovery, protein design), you need a domain-appropriate validity criterion — the framework structure transfers but the hull evaluator does not.
- Agentic orchestrators using LLMs introduce **non-determinism** that is harder to control than fixed pipelines. Even with temperature=0, LLM outputs can vary across API versions.
- The benchmark measures **sample efficiency**, not absolute prediction quality. A method that discovers many candidates just above the stability threshold may score well but produce less useful results than one that finds fewer, more robust candidates.
- Scaling to very high-dimensional search spaces (e.g., quinary+ chemical systems) makes random baselines extremely weak, which can inflate AF/EF metrics and give misleading impressions of method quality.

## Reference

**MADE: Benchmark Environments for Closed-Loop Materials Discovery** — Malik et al., 2026. [arXiv:2601.20996](https://arxiv.org/abs/2601.20996v1). Focus on Section 3 (framework formalization), Algorithm 1 (episode rollout), and Section 5 (ablation results showing component-level contributions to discovery acceleration).