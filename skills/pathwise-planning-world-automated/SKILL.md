---
name: "pathwise-planning-world-automated"
description: "Multi-agent heuristic design framework that uses an entailment graph, policy/world-model/critic agents, and routed reflections to iteratively generate, evolve, and refine algorithmic heuristics for combinatorial optimization and search problems. Use when asked to: 'design a heuristic for TSP', 'optimize a scheduling algorithm', 'evolve a scoring function', 'find a better bin-packing strategy', 'auto-generate a search heuristic', 'improve my greedy algorithm through iteration'."
---

# PathWise: Multi-Agent Heuristic Design via Entailment Graph Planning

This skill enables Claude to apply the PathWise framework for **automated heuristic design** (AHD). Instead of generating a single heuristic and hoping it works, Claude orchestrates three specialized reasoning roles — a policy agent that selects parent heuristics and plans derivation strategies, a world model agent that generates candidate heuristic code, and critic agents that synthesize lessons from successes and failures. All reasoning is anchored to an **entailment graph**: a directed graph where each node is a heuristic (with code, rationale, and performance) and each edge records *how* one heuristic was derived from another. This graph serves as structured memory, preventing redundant exploration and enabling state-aware planning across iterations.

## When to Use

- When the user asks to **design or improve a heuristic function** for a combinatorial optimization problem (TSP, knapsack, vehicle routing, bin packing, scheduling, graph coloring, etc.)
- When the user wants to **iteratively evolve a scoring/priority/evaluation function** and needs a structured search process rather than one-shot generation
- When the user has a **greedy algorithm or metaheuristic template** (e.g., ant colony optimization, local search) and wants Claude to discover the best heuristic component within it
- When the user asks Claude to **compare multiple algorithmic strategies** and systematically derive improvements from the best-performing ones
- When a coding task involves **automated algorithm configuration** — finding the best parameters, priority rules, or decision logic for a solver
- When the user explicitly mentions "PathWise", "automated heuristic design", "evolve a heuristic", or "entailment graph"

## Key Technique

**The core insight** is that heuristic search should not be treated as independent trials. PathWise formulates it as a **sequential decision process (MDP)** over an entailment graph. The state is the current graph of heuristics and their derivation history. The action is selecting parent heuristics plus a natural-language derivation directive (e.g., "combine the distance-based priority of heuristic A with the capacity-awareness of heuristic B, and add a look-ahead penalty"). The transition adds a new heuristic node to the graph. The reward is the measured performance improvement.

**Three agents collaborate per iteration.** The *policy agent* examines the entailment graph — which heuristics exist, which edges worked, which derivation strategies led to improvements — and selects parent nodes plus a derivation rationale. The *world model agent* receives the parent code, descriptions, and rationale, then generates multiple candidate heuristic implementations (rollouts), selecting the best-performing one. Two *critic agents* then provide **routed reflections**: the policy critic analyzes whether the selection strategy was effective ("parent A was a poor choice because its distance metric doesn't generalize"), while the world model critic contrasts the best and worst rollouts ("the top rollout succeeded because it normalized weights; the worst failed by using raw values"). These reflections are injected into subsequent agent prompts as verbal memory.

**Why this matters practically:** Instead of randomly mutating or crossing over heuristics (as in evolutionary approaches), PathWise maintains a causal record of what was tried, what worked, and why. This means Claude can avoid repeating dead-end derivation paths, systematically combine proven sub-strategies, and explain *why* the final heuristic is good — not just *that* it performs well.

## Step-by-Step Workflow

1. **Define the optimization problem and heuristic interface.** Establish the exact function signature the heuristic must implement (inputs, outputs, constraints). For example: `def heuristic(distances: np.ndarray, current_city: int, unvisited: set) -> int` for TSP nearest-neighbor selection. Define the evaluation metric (e.g., total tour length, lower is better).

2. **Generate the initial population (seed heuristics).** Create 3-5 diverse seed heuristics using different algorithmic intuitions: a greedy approach, a randomized approach, a domain-specific rule. Each becomes a root node in the entailment graph with fields: `{code, rationale, description, performance, parents: None}`.

3. **Evaluate all seed heuristics** on a representative set of problem instances. Record the performance metric for each. Annotate each node in the entailment graph with its score.

4. **Policy step: Select parents and plan derivation.** Examine the entailment graph. Identify the top-performing heuristics (exploitation) and any unexplored leaf nodes (exploration). Formulate a derivation directive in natural language, e.g., "Merge heuristic A's nearest-neighbor tie-breaking with heuristic C's look-ahead penalty, and add an adaptive weight that increases penalty for longer remaining tours."

5. **World model step: Generate candidate rollouts.** Using the selected parents' code, descriptions, and the derivation directive, generate 2-4 variant implementations of the new heuristic. Each variant should faithfully follow the directive but may differ in implementation details (data structures, edge cases, numerical constants).

6. **Evaluate rollouts and select the best.** Run all candidate heuristics on the evaluation instances. Insert the best-performing one as a new node in the entailment graph, with directed edges from its parent nodes. Record the derivation rationale on the edge.

7. **Policy critic reflection.** Compare the performance of the new node against its parents. Summarize what the selection strategy got right or wrong: "Selecting parents A and C was effective because both had complementary strengths. However, the derivation directive was too vague about the adaptive weight — next time, specify the decay schedule."

8. **World model critic reflection.** Contrast the best and worst rollouts from step 5: "The best rollout succeeded because it used a normalized distance ratio rather than raw distance. The worst rollout failed because it introduced a division-by-zero when the unvisited set had one element."

9. **Iterate.** Feed both critic reflections into the next policy step as context. Repeat steps 4-8 for 3-8 iterations (or until performance plateaus). Prioritize expanding leaf nodes (heuristics that haven't been used as parents yet) to maintain diversity.

10. **Extract and present the final heuristic.** Select the best-performing node in the entailment graph. Present its code, a natural-language explanation of *why* it works (traced through its derivation path in the graph), and its performance relative to the initial seeds.

## Concrete Examples

**Example 1: Evolving a TSP Nearest-Neighbor Heuristic**

User: "I have a basic nearest-neighbor heuristic for TSP. Can you evolve it into something better?"

Approach:
1. Define interface: `select_next_city(distances, current, unvisited) -> city`
2. Seed the entailment graph with three heuristics:
   - H0: Pure nearest-neighbor (pick closest unvisited city)
   - H1: Nearest-neighbor with random tie-breaking
   - H2: Farthest-insertion (pick the city that maximizes minimum distance to tour)
3. Evaluate on 10 random 50-city instances. Results: H0=gap 18.2%, H1=gap 19.1%, H2=gap 15.7%
4. **Policy step (iteration 1):** Select H0 and H2 as parents. Directive: "Combine nearest-neighbor's greedy local choice with farthest-insertion's global awareness by adding a penalty term proportional to how much selecting a city would isolate remaining unvisited cities."
5. **World model step:** Generate 3 rollouts implementing the directive with different penalty formulations.
6. Best rollout H3: gap 12.4%. Insert into graph with edges from H0, H2.
7. **Policy critic:** "Good parent selection — H0 and H2 had complementary local/global strengths. The penalty concept was productive; refine the isolation metric next."
8. **World model critic:** "Best rollout used average distance to remaining cities as penalty. Worst used max distance, which over-penalized outlier cities."
9. **Iteration 2:** Select H3 as sole parent. Directive: "Add a look-ahead of depth 2: before selecting a city, simulate the next two moves and pick the city that minimizes the 3-step partial tour."
10. H4: gap 9.1%. Final output presented with derivation trace.

Output:
```python
def select_next_city(distances, current, unvisited):
    """Evolved heuristic: nearest-neighbor + isolation penalty + 2-step look-ahead.
    Derivation: H0 (nearest) + H2 (farthest-insertion awareness) -> H3 (isolation penalty)
                -> H4 (added 2-step look-ahead). Gap: 18.2% -> 9.1% over 4 iterations."""
    best_city, best_cost = None, float('inf')
    for city in unvisited:
        remaining = unvisited - {city}
        step1_cost = distances[current][city]
        # 2-step look-ahead
        if remaining:
            lookahead_costs = []
            for next_city in remaining:
                after_next = remaining - {next_city}
                isolation = (sum(distances[next_city][r] for r in after_next) / len(after_next)
                             if after_next else 0)
                lookahead_costs.append(distances[city][next_city] + 0.3 * isolation)
            step2_cost = min(lookahead_costs)
        else:
            step2_cost = 0
        total = step1_cost + 0.7 * step2_cost
        if total < best_cost:
            best_cost, best_city = total, city
    return best_city
```

**Example 2: Designing a Bin-Packing Priority Function**

User: "I need a good priority function for online bin packing. Items arrive one at a time and I need to assign each to a bin."

Approach:
1. Define interface: `score_bin(item_size, bin_remaining_capacity, bin_fill_ratio) -> float` (higher = prefer this bin)
2. Seed heuristics:
   - H0: First-Fit (`return 1.0 if item_size <= capacity else 0.0`)
   - H1: Best-Fit (`return 1.0 / (capacity - item_size + 0.01) if fits else 0.0`)
   - H2: Worst-Fit (`return capacity - item_size if fits else -1`)
3. Evaluate on standard BPP benchmarks (Falkenauer triplet instances, N=500). H0: 1.027 ratio, H1: 1.013, H2: 1.041.
4. **Policy (iter 1):** Parents H1, H0. Directive: "Combine best-fit's tight-packing preference with a bonus for bins that are nearly full (fill_ratio > 0.8), incentivizing completing bins."
5. Best rollout H3: 1.008 ratio. Uses `score = 1/(gap+0.01) + 5.0 * (fill_ratio > 0.8)`.
6. **Policy (iter 2):** Parent H3. Directive: "Replace the hard threshold with a smooth exponential bonus that scales with fill ratio, and add a small penalty for very low fill ratios to avoid scattering items."
7. H4: 1.005 ratio. Final heuristic with smooth exponential weighting.

Output:
```python
def score_bin(item_size, remaining_capacity, fill_ratio):
    """Evolved bin-packing scorer: tight-fit preference + exponential completion bonus.
    Derivation: Best-Fit -> threshold bonus -> smooth exponential. Ratio: 1.013 -> 1.005."""
    if item_size > remaining_capacity:
        return -float('inf')
    gap = remaining_capacity - item_size
    tightness = 1.0 / (gap + 0.01)
    completion_bonus = 3.0 * math.exp(4.0 * (fill_ratio - 0.7))
    scatter_penalty = -0.5 * math.exp(-3.0 * fill_ratio)
    return tightness + completion_bonus + scatter_penalty
```

**Example 3: Evolving a Scheduling Priority Rule**

User: "I'm building a job scheduler. Help me find a good priority function for minimizing total weighted tardiness."

Approach:
1. Interface: `priority(processing_time, due_date, weight, current_time) -> float` (lower = schedule first)
2. Seeds: H0 (EDD: `return due_date`), H1 (WSPT: `return processing_time / weight`), H2 (Slack: `return max(0, due_date - current_time - processing_time)`)
3. Evaluate on 100-job Taillard-style instances. H0: tardiness 847, H1: 623, H2: 712.
4. Iterate 3 times combining WSPT's weight-awareness with slack-based urgency.
5. Final H5: tardiness 489. Combines weighted slack ratio with an urgency multiplier that activates when slack goes negative.

## Best Practices

- **Do:** Maintain the entailment graph explicitly. Track each heuristic's parents, derivation rationale, and performance. This history is what makes PathWise superior to random iteration.
- **Do:** Write derivation directives in natural language that are specific and actionable. "Combine A's distance metric with B's capacity check and normalize by remaining tour length" is far better than "make it better."
- **Do:** Generate multiple rollouts (2-4) per derivation step and pick the best. Single-shot generation misses easy wins from implementation variation.
- **Do:** Use both critic perspectives. The policy critic helps you pick better parents next time; the world model critic helps you write better code next time.
- **Avoid:** Skipping evaluation between iterations. Every new heuristic must be measured on consistent test instances before it can be used as a parent.
- **Avoid:** Always selecting only the top-performing heuristic as parent. Explore leaf nodes and mid-tier heuristics — they may contain sub-strategies that combine well with top performers.
- **Avoid:** Overly long derivation chains without diversity. If performance plateaus after 3 iterations on one branch, start a new branch from a different seed.

## Error Handling

- **Heuristic crashes at runtime:** If a generated heuristic throws an exception on any test instance, assign it the worst possible score and record the failure mode in the world model critic reflection ("rollout failed due to division by zero when bin is empty"). Do not discard it silently — the failure is informative.
- **Performance regression:** If a child heuristic performs worse than all parents, the policy critic should flag the derivation directive as unproductive. Do not add the child to the active frontier; keep it in the graph as a dead-end marker to avoid repeating the strategy.
- **Stale entailment graph:** If after 3+ iterations no improvement is found, inject a new random seed heuristic (using a fundamentally different algorithmic idea) to diversify the search.
- **Evaluation noise:** If the optimization problem is stochastic, evaluate each heuristic on at least 10 instances and use mean performance. A single outlier should not drive parent selection.

## Limitations

- **Requires evaluatable heuristics.** The problem must have a way to score heuristics on test instances programmatically. Purely subjective or human-judged quality functions are not suitable.
- **Computational cost scales with iterations.** Each PathWise iteration involves multiple LLM calls (policy, world model, critics) and multiple evaluations. For problems where a single evaluation takes minutes, limit to 3-5 iterations.
- **Not a replacement for exact solvers.** For small problem instances where optimal solutions are computationally feasible (e.g., TSP with N < 20), use an exact solver. PathWise targets problems where heuristics are necessary due to scale.
- **Heuristic quality depends on seed diversity.** If all seed heuristics encode the same intuition, the entailment graph cannot discover fundamentally different strategies. Start with seeds from different algorithmic families.
- **Single-objective only in basic form.** Multi-objective optimization (Pareto frontiers) requires extending the evaluation and selection logic beyond what is described here.

## Reference

**Paper:** [PathWise: Planning through World Model for Automated Heuristic Design via Self-Evolving LLMs](https://arxiv.org/abs/2601.20539v2) — Gungordu, Xiong, Fekri (2026). Look for: Section 3 (MDP formulation and entailment graph), Section 4 (agent architecture and routed reflections), and Tables 1-3 (performance across COPs and LLM backbones).