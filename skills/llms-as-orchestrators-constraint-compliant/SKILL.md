---
name: "llms-as-orchestrators-constraint-compliant"
description: |
  Build constraint-compliant multi-objective recommendation systems using
  a dual-agent architecture coordinated by an LLM. Implements the DualAgent-Rec
  framework: an Exploitation Agent that optimizes accuracy under hard business
  constraints and an Exploration Agent that searches unconstrained Pareto space
  for diversity, with adaptive epsilon-relaxation guaranteeing 100% constraint
  satisfaction.

  Trigger phrases:
  - "Build a recommendation system with hard constraints"
  - "Multi-objective optimization with constraint satisfaction"
  - "DualAgent-Rec" or "dual-agent recommendation"
  - "Constrained Pareto optimization for recommendations"
  - "Fairness and coverage constraints in recommendations"
  - "LLM-coordinated multi-agent optimization"
---

# LLM-Orchestrated Constraint-Compliant Multi-Agent Recommendation

This skill enables Claude to implement **DualAgent-Rec**, a dual-agent evolutionary
optimization framework for recommendation systems that must satisfy hard business
constraints (e.g., seller diversity, new item exposure, category fairness) while
optimizing multiple competing objectives (relevance, diversity, novelty). The key
insight is splitting optimization into two specialized populations -- one that
respects constraints strictly (exploitation) and one that ignores them (exploration)
-- with an LLM coordinator dynamically allocating resources between them and an
adaptive epsilon-relaxation mechanism guaranteeing feasible final solutions.

## When to Use

- When the user needs a recommendation system that **must never violate** business rules (seller coverage minimums, fairness quotas, new-item exposure thresholds)
- When building multi-objective recommenders that balance relevance, diversity, and novelty simultaneously
- When the user asks for Pareto-optimal recommendation lists rather than single-score rankings
- When existing recommendation pipelines treat constraints as soft penalties and suffer production violations
- When the user wants an LLM to orchestrate or coordinate multiple optimization agents adaptively
- When implementing evolutionary multi-objective optimization (NSGA-II style) with constraint handling in Python

## Key Technique

**Dual-agent decomposition.** Instead of a single population that tries to satisfy constraints and optimize objectives simultaneously, DualAgent-Rec maintains two separate populations. The **Exploitation Agent** uses the Constraint Domination Principle (CDP): feasible solutions always dominate infeasible ones, and among infeasible solutions, lower total constraint violation wins. It uses DE/pbest/1 mutation to converge quickly within feasible regions. The **Exploration Agent** deliberately ignores all constraints, applying unconstrained Pareto dominance with doubled mutation rate to discover high-quality solutions in otherwise unreachable regions of objective space.

**LLM coordination and epsilon-relaxation.** Every T generations (default: 10), the LLM coordinator observes optimization state (hypervolume, improvement rates, feasibility ratios, violation magnitudes) and outputs an allocation ratio alpha in [0, 1] that resizes the two populations. Early in optimization, alpha favors exploration (~0.60); as convergence progresses, it shifts toward exploitation (~0.80); if constraint violations spike, it drops back (~0.55). Meanwhile, the epsilon-relaxation mechanism starts permissive (epsilon_0 from the 80th percentile of initial violations) and decays exponentially: `epsilon_t = epsilon_0 * 0.8^(t / T_max)`. This lets early generations explore infeasible space freely while guaranteeing 100% feasibility at termination.

**Elite knowledge transfer.** After each generation, the top solutions by crowding distance from both populations are exchanged. High crowding distance means sparsely populated Pareto front regions, so transfers inject diversity rather than redundancy. This bidirectional exchange is what makes the dual-agent split strictly better than a single-population approach.

## Step-by-Step Workflow

1. **Define objectives and constraints formally.** Enumerate each objective as a maximization function (relevance, diversity, novelty) and each hard constraint as `g_j(L) <= 0`. Use Gini coefficient for category fairness, distinct-seller count for seller coverage, and recency ratio for new-item exposure. Store thresholds in a config dict.

2. **Encode recommendation lists as individuals.** Represent each candidate list as a fixed-length integer array of item indices (default k=10). Initialize two populations of size N/2 each (default N=100 total) by sampling items from the catalog weighted by base relevance scores.

3. **Implement the three objective functions.**
   - Relevance: category overlap between list and user history + cosine similarity from pre-trained embeddings.
   - Diversity: mean pairwise cosine distance across list items: `f2 = (2 / k(k-1)) * sum(d(x_i, x_j))`.
   - Novelty: mean inverse popularity: `f3 = (1/k) * sum(1 - pop(i))`.

4. **Implement constraint evaluation.** For each list, compute all `g_j(L)` values. Total constraint violation `CV(L) = sum(max(0, g_j(L)))`. Initialize `epsilon_0` from the 80th percentile of CV values across the initial combined population.

5. **Build the Exploitation Agent.** Each generation: select parents via tournament selection using CDP ordering (feasible > infeasible; lower CV > higher CV; Pareto dominance among feasible). Apply DE/pbest/1 mutation: `v = x_pbest + F * (x_r1 - x_r2)` where `x_pbest` is drawn from the top 10% of feasible solutions. Enforce epsilon-relaxed constraints: treat a solution as feasible if `CV(L) <= epsilon_t`.

6. **Build the Exploration Agent.** Each generation: select parents via standard NSGA-II non-dominated sorting and crowding distance, ignoring all constraints. Use polynomial mutation with doubled rate (pm=0.2 instead of 0.1) and SBX crossover to encourage broad search.

7. **Implement elite transfer.** After both agents evolve, merge populations, compute crowding distances on the combined non-dominated front, and transfer the top `ceil(0.1 * N)` solutions by crowding distance into each agent's population (replacing worst members).

8. **Implement the LLM coordinator.** Every T=10 generations, construct a structured prompt containing: current generation, each agent's hypervolume, improvement rate over last T generations, feasibility ratio, mean violation magnitude, and current alpha. The LLM returns a new alpha value. Parse it and resize populations: `|P_exploit| = floor(alpha * N)`, `|P_explore| = N - |P_exploit|`.

9. **Decay epsilon and iterate.** At each generation t, compute `epsilon_t = epsilon_0 * 0.8^(t / T_max)`. Run for T_max=50 generations. By the final generation, epsilon is near zero, enforcing strict constraint satisfaction.

10. **Select final output.** From the exploitation agent's final population, extract all fully feasible solutions (CV=0). Among those, select the solution with highest hypervolume contribution or apply a user-specified preference (e.g., weighted sum) to pick a single recommendation list.

## Concrete Examples

**Example 1: E-commerce product recommendations with seller diversity**

User: "Build a recommendation system for our marketplace. Each list of 10 items must include products from at least 3 different sellers and at least 1 item listed in the last 30 days. Optimize for relevance and diversity."

Approach:
1. Define constraints: `g_seller(L) = 3 - num_distinct_sellers(L) <= 0`, `g_new(L) = 1 - count_new_items(L) <= 0`
2. Define objectives: relevance (embedding similarity to user history), diversity (intra-list cosine distance)
3. Initialize dual populations from item catalog with seller/date metadata
4. Run 50-generation optimization with epsilon-relaxation starting at 80th percentile violation
5. LLM coordinator shifts alpha from 0.60 to 0.80 over the run

Output:
```python
# Final recommendation for user_42
recommendation = {
    "items": ["SKU-1021", "SKU-3847", "SKU-0512", ...],  # 10 items
    "sellers": ["seller_A", "seller_B", "seller_C", "seller_A", ...],  # 4 distinct sellers (>=3)
    "new_items": ["SKU-0512"],  # 1 new item (>=1)
    "relevance_score": 0.74,
    "diversity_score": 0.31,
    "constraint_violation": 0.0,  # all constraints satisfied
}
```

**Example 2: Content feed with fairness constraints**

User: "Our news feed recommends 20 articles. We need the Gini coefficient across content categories to stay below 0.5, and at least 2 articles must be from underrepresented creators. Maximize engagement prediction and topic diversity."

Approach:
1. Define constraints: `g_gini(L) = gini(category_distribution(L)) - 0.5 <= 0`, `g_underrep(L) = 2 - count_underrep(L) <= 0`
2. Set k=20, adjust population size to N=150 for larger solution space
3. Exploitation agent uses CDP to enforce Gini and underrepresented-creator constraints
4. Exploration agent finds diverse article combinations ignoring fairness rules
5. Elite transfer injects high-diversity solutions into exploitation population

Output:
```python
# Pareto front summary (3 representative solutions)
pareto_front = [
    {"engagement": 0.82, "diversity": 0.19, "gini": 0.43, "underrep": 3},  # accuracy-focused
    {"engagement": 0.71, "diversity": 0.34, "gini": 0.38, "underrep": 4},  # balanced
    {"engagement": 0.63, "diversity": 0.41, "gini": 0.31, "underrep": 5},  # diversity-focused
]
# All solutions: gini < 0.5, underrep >= 2
```

**Example 3: Implementing the LLM coordinator prompt**

User: "Show me how to build the LLM coordinator that decides the exploitation/exploration ratio."

Approach:
1. Collect optimization state every 10 generations
2. Format a structured prompt with metrics from both agents
3. Parse the LLM response for the alpha value

Output:
```python
def build_coordinator_prompt(gen, exploit_metrics, explore_metrics, epsilon_t, alpha_current):
    return f"""You are coordinating a dual-agent recommendation optimizer.

Current state (generation {gen}/50):
- Exploitation agent: HV={exploit_metrics['hv']:.4f}, improvement_rate={exploit_metrics['delta_hv']:.4f}, feasibility={exploit_metrics['feasibility']:.1%}
- Exploration agent: HV={explore_metrics['hv']:.4f}, improvement_rate={explore_metrics['delta_hv']:.4f}
- Current epsilon relaxation: {epsilon_t:.4f}
- Current allocation alpha: {alpha_current:.2f} (exploitation share)

Rules:
- If feasibility < 90%, decrease alpha by 0.10-0.15 to boost exploration
- If exploitation improvement_rate < 0.001, increase exploration temporarily
- In later generations (>35), bias toward exploitation (alpha >= 0.75)
- Alpha must be between 0.3 and 0.9

Return ONLY a JSON object: {{"alpha": <float>}}"""

def parse_coordinator_response(response_text):
    import json, re
    match = re.search(r'\{[^}]+\}', response_text)
    if match:
        result = json.loads(match.group())
        return max(0.3, min(0.9, result["alpha"]))
    return 0.7  # safe fallback
```

## Best Practices

- **Do:** Formulate all business rules as hard constraints (`g_j(L) <= 0`) from the start. Never encode must-have rules as objective penalties -- that is the core failure mode this framework solves.
- **Do:** Initialize epsilon_0 from actual constraint violation statistics of your initial population (80th percentile), not an arbitrary value. This calibrates relaxation to your specific problem scale.
- **Do:** Use crowding-distance-based elite transfer rather than random exchange. This preserves Pareto front spread and prevents population collapse.
- **Do:** Set the LLM coordination interval T proportional to population size (T=10 for N=100). Too frequent coordination wastes compute; too infrequent misses constraint violation spikes.
- **Avoid:** Letting the exploration agent influence final solution selection. Always select final outputs exclusively from the exploitation agent's feasible set.
- **Avoid:** Skipping the dual-agent split for "simple" constraint sets. Even one hard constraint benefits from the exploration agent discovering high-quality solutions near constraint boundaries that pure feasibility-first search misses.

## Error Handling

- **No feasible solutions found by final generation:** This means constraints are likely mutually exclusive given the catalog. Verify constraint thresholds are achievable by checking a brute-force sample. If constraints are too tight, surface this to the user with the minimum violation achieved rather than returning infeasible results silently.
- **LLM coordinator returns unparseable output:** Always implement a fallback alpha (0.7) and validate that alpha is within [0.3, 0.9]. Log coordinator failures for monitoring but never let them crash the optimization loop.
- **Population convergence (all individuals identical):** Inject random immigrants (5-10% of population) when population diversity drops below a threshold. The exploration agent's doubled mutation rate mitigates this, but small catalogs can still converge prematurely.
- **Epsilon doesn't reach zero:** With gamma=0.8 and T_max=50, `epsilon_50 = epsilon_0 * 0.8^1 = 0.8 * epsilon_0`. For stricter convergence, use gamma=0.95 with more generations or add a hard cutoff: `epsilon_t = 0` for `t >= T_max - 5`.
- **Slow runtime for large catalogs:** The O(2N*M) complexity means ~2x overhead vs single-population. Reduce N or T_max for real-time use cases, or pre-filter the catalog to a candidate set of 500-1000 items before evolutionary search.

## Limitations

- The evolutionary optimization requires 50 generations over a population of 100, taking ~35 seconds per user on commodity hardware. This is suitable for batch/offline recommendations but not sub-100ms real-time serving without pre-computation.
- The LLM coordinator adds interpretability but only ~1% quantitative improvement over a fixed rule-based alpha schedule. For latency-sensitive deployments, a rule-based coordinator (`alpha = 0.6 + 0.4 * t/T_max`) achieves nearly identical results without LLM calls.
- Validated on Amazon Reviews 2023 with up to ~4,500 items per category. Catalogs with millions of items require a candidate generation stage (e.g., embedding retrieval) before applying DualAgent-Rec to the top-N candidates.
- The framework assumes objectives and constraints can be evaluated per-list without expensive external calls. If constraint checking requires database lookups per evaluation, the 100-population x 50-generation loop becomes a bottleneck.

## Reference

**Paper:** [LLMs as Orchestrators: Constraint-Compliant Multi-Agent Optimization for Recommendation Systems](https://arxiv.org/abs/2601.19121v3) (Zhang et al., 2026)

Key sections to reference: Section 3 for the full DualAgent-Rec algorithm with pseudocode, Section 3.3 for the adaptive epsilon-relaxation formula, Table 5 for LLM coordinator allocation patterns across optimization phases, and Section 4 for experimental validation on Amazon Reviews 2023.