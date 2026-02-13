---
name: "deltaevolve-accelerating-scientific-discovery"
description: "Iteratively evolve code solutions using momentum-driven semantic deltas instead of full-code histories. Use when: 'evolve a better heuristic for bin packing', 'optimize this algorithm iteratively', 'use LLM-driven evolution to improve this function', 'find a better solution through evolutionary search', 'iteratively refine this solver', 'apply DeltaEvolve to discover a better algorithm'."
---

# DeltaEvolve: Momentum-Driven Evolutionary Code Optimization

This skill enables Claude to iteratively evolve code solutions toward optimal performance using the DeltaEvolve framework. Instead of tracking full code snapshots between iterations (as in AlphaEvolve/FunSearch), DeltaEvolve captures **structured semantic deltas** -- concise descriptions of *what changed and why* -- between successive program versions. These deltas accumulate into a momentum vector that steers the LLM toward productive search directions while consuming ~37% fewer tokens than full-code approaches. The technique applies to any problem where a candidate program can be scored by an evaluator: heuristic design, symbolic regression, optimization solvers, algorithm discovery, and more.

## When to Use

- When the user asks to **iteratively improve a function or algorithm** by trying variations and keeping what works (e.g., "evolve a better priority function for this scheduler")
- When the user wants to **discover heuristics** for combinatorial problems like bin packing, graph coloring, or scheduling
- When the user has a **scoring/evaluation function** and wants to search for the best candidate program that maximizes it
- When the user asks to **optimize a numerical solver, PDE discretization, or regression formula** through evolutionary search
- When the user wants to **automatically explore algorithm design space** rather than manually tuning code
- When the user mentions "FunSearch", "AlphaEvolve", or "LLM-driven evolution" and wants a structured approach

## Key Technique

DeltaEvolve formalizes LLM-driven evolution as an **Expectation-Maximization loop**. In the E-step, the LLM samples candidate programs by applying delta-based modifications to a parent solution. In the M-step, the system evaluates candidates, updates a database of scored deltas, and selects context for the next iteration. The critical insight is that the *optimizable context* -- what the LLM sees in its prompt -- matters far more than scalar fitness scores. Ablation studies show that removing numerical scores barely hurts performance, but random context selection causes collapse. This means the system works primarily through **in-context learning from well-chosen examples**, not regression on scores.

The core innovation is replacing full-code histories with a **three-level delta representation**. Level 1 is a one-line summary ("FROM: greedy-by-weight TO: greedy-by-density-with-lookahead", ~30 tokens). Level 2 is a structured delta plan listing each modified component with old logic, new logic, and the hypothesis behind the change. Level 3 is the full executable code. A **progressive disclosure mechanism** feeds older iterations as Level-1 summaries ("ancient history"), recent iterations as Level-2 plans ("recent insights"), and only the parent node as Level-3 full code ("immediate context"). This mirrors how momentum in SGD accumulates gradient differences (x_i - x_{i-1}): the semantic deltas capture program differences (p_i - p_{i-1}) and their accumulated trajectory encodes the prevailing direction of improvement.

The population is managed via an **island model** with multiple subpopulations. Parent selection is stochastic but biased toward high-reward nodes. Elite nodes (top-k scorers) and diverse nodes (selected via a MAP-Elites grid over behavioral features) are included in context to balance exploitation and exploration.

## Step-by-Step Workflow

1. **Define the problem interface.** Write a `solve(input) -> output` function signature and an `evaluate(output) -> float` scoring function. The scoring function must be deterministic and fast -- it runs on every candidate. Clearly specify what "higher is better" means.

2. **Create the seed program.** Write a simple baseline implementation of `solve()`. This can be a naive greedy heuristic, a random approach, or even a stub. It must be runnable and produce a valid score. Store this as iteration 0 in the evolution database.

3. **Initialize the delta database.** Create a structured store (list or dict) with entries of the form:
   ```
   {
     "iteration": 0,
     "level1_summary": "Baseline: naive greedy by first-fit",
     "level2_plan": [],
     "code": "<full source of seed>",
     "score": <baseline_score>,
     "parent": null
   }
   ```

4. **Select context nodes for the prompt.** For each new iteration, pick:
   - **Parent node**: stochastic selection biased toward high scores (e.g., softmax over scores with temperature)
   - **Elite nodes** (top-k=3): highest-scoring solutions in the database
   - **Diverse nodes** (m=2): solutions that differ most from elites (measured by edit distance on Level-1 summaries or behavioral features like output distribution)

5. **Construct the evolution prompt with progressive disclosure.** Build the LLM prompt as:
   ```
   ## Problem: <problem description and evaluation criteria>
   ## Ancient History (Level 1 summaries):
   - Iter 2: FROM greedy-first-fit TO greedy-best-fit | Score: 0.72 -> 0.78
   - Iter 5: FROM greedy-best-fit TO density-aware-best-fit | Score: 0.78 -> 0.83
   ## Recent Insights (Level 2 delta plans):
   - Iter 8: Changed bin selection from min-remaining to max-density.
     Hypothesis: denser packing leaves larger contiguous gaps. Result: +0.03
   ## Current Parent (Level 3 full code):
   <complete source code of parent, score=0.86>
   ## Elite Scores: [0.91, 0.89, 0.86]
   ## Task: Generate a new variant. Output a delta summary, delta plan, then full code.
   ```

6. **Generate a candidate.** Invoke the LLM with the constructed prompt. Parse the response into three levels using delimiters:
   - `#Delta-Summary-Start` ... `#Delta-Summary-End` -> Level 1
   - `#Delta-Plan-Start` ... `#Delta-Plan-End` -> Level 2
   - `#Code-Start` ... `#Code-End` -> Level 3 (executable code)

7. **Evaluate the candidate.** Run the extracted code through the evaluation function. Record the score. Compute the delta: `delta_score = new_score - parent_score`. Tag the delta as "Improved" or "Degraded".

8. **Update the database.** Store the new node with all three levels, the score, and a link to its parent. If the candidate improves on the parent, it becomes eligible for future parent selection. If it degrades, it still enters the database (negative results inform future context selection by showing what *not* to do).

9. **Repeat for N iterations.** Run steps 4-8 for a budget of iterations (typically 20-100). Monitor for convergence: if the top score hasn't improved in 10 iterations, increase diversity selection weight or reset one island with a fresh random seed.

10. **Extract the best solution.** Return the highest-scoring program from the database along with its full ancestry chain of Level-1 summaries, which serves as a human-readable explanation of how the solution was discovered.

## Concrete Examples

**Example 1: Evolving a Bin Packing Heuristic**

User: "I have an online bin packing problem. Items arrive one at a time with sizes in (0,1]. I need a function that decides which bin to place each item in. Help me evolve a good heuristic."

Approach:
1. Define interface: `place_item(item_size, bins) -> bin_index` and evaluator that measures total bins used on 1000 random instances.
2. Seed with first-fit decreasing as baseline. Score: uses 312 bins on average.
3. Iterate with DeltaEvolve for 30 rounds:
   - Iter 1 delta: "FROM first-fit TO best-fit" -> 298 bins (Improved)
   - Iter 5 delta: "FROM best-fit TO best-fit-with-3-item-lookahead" -> 287 bins (Improved)
   - Iter 12 delta: "FROM lookahead TO hybrid-density-score weighting remaining capacity and item frequency" -> 271 bins (Improved)
   - Iter 18 delta: "FROM hybrid-density TO adaptive-threshold that switches strategy when bin utilization > 85%" -> 264 bins (Improved)
4. Return best solution at iteration 18 with full delta ancestry.

Output:
```python
def place_item(item_size, bins):
    """Evolved heuristic: adaptive threshold with density scoring.
    Discovery path: first-fit -> best-fit -> lookahead -> density -> adaptive-threshold
    """
    THRESHOLD = 0.85
    best_bin, best_score = -1, float('inf')
    for i, remaining in enumerate(bins):
        if remaining >= item_size:
            utilization = 1.0 - remaining
            if utilization > THRESHOLD:
                score = remaining - item_size  # tight packing mode
            else:
                score = abs(remaining - item_size * 2.2)  # density mode
            if score < best_score:
                best_score, best_bin = score, i
    if best_bin == -1:
        bins.append(1.0 - item_size)
        return len(bins) - 1
    bins[best_bin] -= item_size
    return best_bin
```

**Example 2: Symbolic Regression**

User: "I have (x, y) data from a physics experiment. Find a compact formula that fits it. Use evolutionary search."

Approach:
1. Define interface: `formula(x) -> y` and evaluator = negative mean squared error on held-out data.
2. Seed with `y = x` (MSE = 14.3).
3. Evolve for 25 rounds with DeltaEvolve:
   - Iter 1: "FROM linear TO quadratic" -> MSE 8.1
   - Iter 4: "FROM quadratic TO sin-modulated quadratic" -> MSE 3.2
   - Iter 9: "FROM sin-modulated TO damped oscillator with exponential envelope" -> MSE 0.41
   - Iter 15: "FROM damped oscillator TO damped oscillator with phase-shifted harmonic" -> MSE 0.08

Output:
```python
import numpy as np

def formula(x):
    """Discovered: damped oscillator with phase shift.
    y = 2.31 * exp(-0.15*x) * sin(1.87*x + 0.42) + 0.73*x
    """
    return 2.31 * np.exp(-0.15 * x) * np.sin(1.87 * x + 0.42) + 0.73 * x
```

**Example 3: Optimizing a Sort Comparator**

User: "I have a custom sort for ranking search results with multiple signals (relevance, recency, popularity). Evolve a better ranking function."

Approach:
1. Define evaluator: NDCG@10 against human-labeled relevance judgments on a test set.
2. Seed with `score = 0.5*relevance + 0.3*recency + 0.2*popularity`. NDCG = 0.71.
3. Evolve for 20 rounds:
   - Iter 3: "FROM linear-combo TO log-scaled-popularity with relevance floor" -> NDCG 0.74
   - Iter 8: "FROM log-scaled TO nonlinear interaction between recency and relevance" -> NDCG 0.78
   - Iter 14: "FROM nonlinear TO piecewise: boost recent+relevant, penalize old+popular-only" -> NDCG 0.82

Output: the best-scoring ranking function with its delta ancestry as documentation.

## Best Practices

- **Do:** Keep the evaluator fast and deterministic. Every candidate must be scored, so evaluation time dominates wall-clock cost. Use representative subsets if full evaluation is slow.
- **Do:** Include both elite (high-scoring) and diverse (structurally different) nodes in the prompt context. Diversity prevents the search from collapsing to a local optimum.
- **Do:** Preserve negative results in the database. Showing the LLM "this change degraded performance" is as informative as showing improvements -- it prunes the search space.
- **Do:** Use the three-level progressive disclosure strictly. Old iterations get Level-1 summaries only. This keeps prompt size bounded even after 100+ iterations.
- **Avoid:** Feeding the LLM raw scores without context about what changed. The ablation evidence shows scores alone contribute almost nothing -- the *selected context* does the work.
- **Avoid:** Running too many iterations without diversity injection. If top score plateaus for 10+ iterations, introduce a random restart or pull from a different island.
- **Avoid:** Making the seed program too complex. A simple baseline gives the evolutionary process room to discover structure. Starting with an overengineered solution constrains the search.

## Error Handling

| Issue | Symptom | Resolution |
|-------|---------|------------|
| Generated code doesn't parse | Syntax errors in extracted code block | Re-prompt with the error message appended. Use stricter delimiters. Fall back to parent. |
| Score regression across all candidates | Every candidate in a batch scores lower than parent | Widen diversity selection. Include more elite nodes. Reduce mutation aggressiveness by showing more Level-2 detail. |
| Evaluation timeout | Candidate contains infinite loops or exponential complexity | Wrap evaluation in a timeout. Assign worst-possible score to timed-out candidates. Add "must terminate in O(n log n)" to prompt constraints. |
| Prompt exceeds context window | Too many history nodes selected | Aggressively compress: show only Level-1 for all but the 3 most recent nodes. Reduce elite/diverse counts. |
| Premature convergence | All candidates become near-copies of the best solution | Reset one island with a random or adversarial seed. Temporarily increase the temperature of parent selection. |
| Delta parse failure | LLM doesn't produce delimited output | Retry with explicit few-shot examples of the delimiter format. If persistent, extract delta post-hoc by diffing parent and child code. |

## Limitations

- **Requires a fast, automated evaluator.** If scoring a candidate takes minutes or requires human judgment, the iterative loop becomes impractical. This rules out problems where quality is subjective or evaluation is expensive.
- **LLM context window bounds history depth.** Even with progressive disclosure, very long evolution chains (500+ iterations) may require aggressive pruning of the database, losing potentially useful historical signal.
- **No formal convergence guarantees.** Unlike gradient descent, the momentum analogy is qualitative. The method can stall, oscillate, or miss global optima -- it is a heuristic search, not an optimizer with provable rates.
- **Sensitive to problem decomposability.** DeltaEvolve works best when solutions are modular and small changes have measurable effects. Monolithic algorithms where every line interacts with every other line yield noisy, unhelpful deltas.
- **Single-file scope.** The technique is designed for evolving a single function or module. Multi-file system design or architectural search is out of scope.

## Reference

**Paper:** [DeltaEvolve: Accelerating Scientific Discovery through Momentum-Driven Evolution](https://arxiv.org/abs/2602.02919v1) by Jiachen Jiang, Tianyu Ding, Zhihui Zhu (2026). Look for: the EM formalization (Section 3), the three-level delta database (Section 4.2), progressive disclosure rendering (Section 4.3), and ablation results in Table 1 showing context selection dominates over scalar feedback.