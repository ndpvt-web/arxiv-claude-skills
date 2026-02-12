---
name: "internalizing-reasoning-discovery-replay"
description: "Apply the STIR framework (Self-Distilled Tools for Internal Reasoning) to structure multi-step reasoning tasks using latent action discovery, curation, and replay. Mine successful reasoning sub-paths from exploratory attempts, build a compact library of reusable reasoning primitives, and dynamically inject them to steer future problem-solving. Triggers: 'optimize my reasoning pipeline', 'internalize chain-of-thought', 'discover reasoning patterns', 'build a reasoning tool library', 'reduce verbose reasoning overhead', 'self-distill reasoning strategies'"
---

# Internalizing Reasoning via Discovery and Replay of Latent Actions

This skill teaches Claude to apply the STIR (Self-Distilled Tools for Internal Reasoning) framework from Shi et al. (2026) to real coding and problem-solving tasks. The core idea: instead of generating long explicit chains of thought every time, you **mine successful reasoning sub-paths** from multiple exploratory attempts, **curate them into a compact library of reusable reasoning primitives**, and **replay the best-matching primitive** when facing a new problem. This converts verbose, redundant reasoning into efficient, targeted interventions — achieving better accuracy with fewer tokens.

## When to Use This Skill

- When building an agent or pipeline that solves multi-step reasoning tasks (math, code generation, logic puzzles) and you want to reduce inference cost without losing accuracy
- When you have a dataset of problems where some rollouts succeed and others fail, and you want to extract *what made the difference* between success and failure
- When the user asks to "optimize chain-of-thought", "reduce reasoning tokens", or "make my agent think more efficiently"
- When designing a retrieval-augmented reasoning system that selects and applies reasoning strategies dynamically based on problem context
- When implementing self-distillation: training a model or agent on its own successful exploration traces
- When the user wants to build a "reasoning tool library" — a reusable set of problem-solving heuristics extracted from past performance

## Key Technique

**The core insight:** Successful and failed reasoning attempts diverge at specific decision points. By computing the vector difference between centroids of successful vs. failed hidden states at those decision points, you obtain a *steering primitive* — a compact representation of the implicit correction needed to get back on the right track. STIR collects many such primitives, prunes them down to a diverse library, and retrieves + applies the right one at inference time.

**Three-stage pipeline:**

1. **Discovery** — Sample multiple reasoning rollouts per problem. Score them with a length-penalized reward (correct answer gets credit, verbosity gets penalized). At each structural checkpoint in the reasoning trace, partition rollouts into high-reward and low-reward groups. Compute the centroid difference: `v = centroid(successes) - centroid(failures)`. This vector `v` is the steering impulse — the direction that moves reasoning from failure toward success. Store both correction entries (keyed by failure centroids, linked to `v`) and anchor entries (keyed by success centroids, linked to null — meaning "no intervention needed here").

2. **Curation** — The raw set of discovered primitives is redundant. Apply a Quality-Diversity selection (greedy DPP) that simultaneously maximizes quality scores and geometric diversity. This yields ~256 primitives from potentially thousands of candidates — a compact, orthogonal tool library covering distinct reasoning failure modes.

3. **Replay** — At inference time, extract the current reasoning state, retrieve the top-k matching primitives by cosine similarity, check anchor gating (if the current state already matches a "success" pattern, abstain from intervention), run a short lookahead probe to evaluate each candidate's effect, then inject the best-scoring primitive with adaptive strength. The injection is a simple vector addition to the current state, clipped to prevent over-steering.

**Why this matters for practical coding tasks:** The framework generalizes beyond neural hidden states. Any system where you can represent "reasoning state" as a vector (embeddings of code context, intermediate AST features, test-result encodings) can use the discover-curate-replay loop to build reusable problem-solving strategies from past exploration.

## Step-by-Step Workflow

1. **Define the problem space and reward signal.** Identify the class of tasks (e.g., code debugging, math word problems, SQL generation). Define a binary correctness signal plus a verbosity penalty: `R = correctness - eta * (token_count / max_tokens)`. This ensures discovered primitives favor concise correct solutions over verbose ones.

2. **Generate diverse rollouts.** For each problem in your training set, sample K independent reasoning traces (K=8 is a good default). Use temperature sampling or nucleus sampling to ensure diversity. Record the full trace including intermediate states at structural checkpoints (paragraph breaks, function boundaries, logical transitions).

3. **Partition rollouts at each checkpoint.** At every structural boundary in the reasoning trace, split the K rollouts into a positive set (above-median reward) and negative set (below-median reward). Require a minimum gap between the sets to ensure meaningful signal — skip checkpoints where all rollouts are equally good or bad.

4. **Extract steering primitives.** For each valid checkpoint: compute the centroid embedding of positive-set prefixes (`mu+`) and negative-set prefixes (`mu-`). The steering impulse is `v = mu+ - mu-`. Store two memory entries: a *correction entry* keyed by `mu-` with impulse `v`, and an *anchor entry* keyed by `mu+` with null impulse. The anchor entries prevent unnecessary intervention on already-correct reasoning paths.

5. **Curate the primitive library via Quality-Diversity selection.** From all extracted primitives, run greedy DPP selection: iteratively pick the primitive that maximizes `log(1 + quality_score) + lambda * log(conditional_variance)`. The quality term favors high-reward primitives; the variance term favors primitives that are geometrically distinct from those already selected. Target a library size of 128-256 primitives. L2-normalize all keys for cosine retrieval.

6. **Implement the retrieval-and-replay engine.** At inference time, encode the current reasoning state into the same embedding space as your library keys. Retrieve top-k candidates (k=8) by cosine similarity. Check for anchor dominance: if most retrieved entries are anchors, the current state is already on a good path — abstain from intervention.

7. **Run lookahead probes to validate candidates.** For each non-anchor candidate, simulate a short continuation (4 tokens) with and without the steering impulse applied. Compute the gain as the average log-probability improvement. This prevents applying primitives that looked relevant by key similarity but would actually degrade output quality.

8. **Score and inject the best primitive.** Compute a unified score: `S = beta * retrieval_similarity + rho * lookahead_gain`. If the top score exceeds a null-action threshold, inject the primitive: `state = state + clip(scale * S, 0, alpha_max) * impulse_vector`. Otherwise, abstain. The adaptive strength clipping prevents catastrophic over-steering.

9. **Evaluate and iterate.** Measure accuracy and token efficiency on a held-out test set. Compare against baseline (no intervention) and static approaches (single fixed steering vector). Tune the null-action threshold — too low causes over-intervention, too high makes the system too conservative. Good defaults: `beta=2.0, rho=0.1, tau_null=0.3`.

10. **Transfer and generalize.** Test the library on related but unseen task distributions. STIR primitives often transfer across tasks in the same domain (e.g., AIME math tools work on AMC problems). Expand the library incrementally as you encounter new failure modes.

## Concrete Examples

**Example 1: Building a Reasoning Primitive Library for Code Debugging**

```
User: I have an agent that debugs Python code. It generates long reasoning
traces but often goes in circles. Help me build a reasoning tool library
using the STIR approach to make it more efficient.

Approach:
1. Collect 200 buggy Python programs with known fixes. For each, run the
   debug agent 8 times with temperature=0.7, recording the full reasoning
   trace and whether it found the correct fix.

2. At each reasoning boundary (after each "hypothesis" or "test" step),
   embed the trace prefix using a code-aware encoder (e.g., CodeBERT or
   the agent's own internal representation).

3. Partition into success/failure sets at each boundary. Extract steering
   vectors: v = centroid(successful_prefixes) - centroid(failed_prefixes).
   Store correction entries (keyed by failure centroid) and anchor entries
   (keyed by success centroid).

4. From ~1200 raw primitives, apply QD-DPP to select 200 diverse entries.
   Common patterns that emerge:
   - "Check types before logic" primitive (keyed to states where the agent
     was reasoning about control flow but the bug was a type error)
   - "Read the error message literally" primitive (keyed to states where
     the agent was speculating instead of parsing the traceback)
   - "Isolate the failing line" primitive (keyed to states where the agent
     was analyzing the whole function instead of narrowing scope)

5. At debug time: encode current reasoning state, retrieve top-8 from
   library, check anchor gating, probe top candidates, inject the winner.

Output:
- Library of 200 curated debug-reasoning primitives stored as JSON:
  { "key": [0.12, -0.34, ...], "impulse": [0.05, 0.11, ...],
    "quality": 0.87, "label": "type-check-first", "is_anchor": false }
- Inference wrapper that queries the library at each reasoning step
- Expected: 20-35% fewer reasoning tokens, 3-7% accuracy improvement
```

**Example 2: Self-Distilling Math Problem-Solving Strategies**

```
User: I want to extract reusable math reasoning strategies from a model's
own successful attempts and replay them on harder problems.

Approach:
1. Take 500 competition math problems (AMC/AIME level). For each, generate
   8 solution attempts. Score: R = correct - 0.1*(tokens/max_tokens).

2. At each paragraph break in the solution, partition attempts into
   high-reward (top 4) and low-reward (bottom 4) sets. Compute centroid
   difference vectors in the model's embedding space.

3. Curate to 256 primitives using greedy QD-DPP (lambda=0.5). Typical
   discovered primitives:
   - "Substitute small values" (fires when model is stuck on general case)
   - "Convert to coordinates" (fires when model is reasoning abstractly
     about geometry)
   - "Factor the expression" (fires when model is expanding instead of
     simplifying)

4. At test time on new problems: retrieve → gate → probe → inject.
   Use beta=2.0, rho=0.1, k_scale=0.75, tau_null=0.3.

Output:
- Primitive library file: math_reasoning_tools.json (256 entries)
- Evaluation on held-out MATH-500: +4.2% accuracy, -28% token usage
- Cross-domain transfer test on ARC-Challenge: +1.9% accuracy with
  zero additional training
```

**Example 3: Reducing Agent Verbosity in Multi-Step Planning**

```
User: My task-planning agent writes extremely verbose plans. I want to
keep accuracy but cut the reasoning length by a third.

Approach:
1. Collect 300 planning tasks with ground-truth solutions. Generate 8
   plans each. Score with correctness AND length penalty (eta=0.15).

2. The length penalty ensures high-reward rollouts are both correct AND
   concise. Steering primitives now point toward concise-correct states,
   away from verbose-correct or verbose-incorrect states.

3. At discovery time, focus on checkpoints where traces diverge in length:
   same correctness but different verbosity. The impulse vectors capture
   "skip unnecessary elaboration" patterns.

4. Curate 128 primitives. Apply during planning with conservative
   threshold (tau_null=0.4) to avoid cutting essential reasoning.

Output:
- Planning stays at 97% accuracy (was 98% — within noise)
- Token usage drops by 33% on average
- Largest gains on routine sub-problems where the agent was over-explaining
  well-known patterns
```

## Best Practices

- **Do:** Use a length-penalized reward signal (`R = correctness - eta * normalized_length`) when discovering primitives. Without the length penalty, you optimize only for correctness and miss the efficiency gains that make STIR valuable.
- **Do:** Store anchor entries (success-keyed, null-impulse) alongside correction entries. Anchors prevent the system from intervening when reasoning is already on track — this is critical for avoiding regression on easy problems.
- **Do:** Apply the QD-DPP curation step even if your raw primitive set seems manageable. Redundant primitives cause retrieval confusion and degrade lookahead probe efficiency. Target a library 10-20x smaller than the raw set.
- **Do:** Tune the null-action threshold (`tau_null`) on a validation set. This single parameter controls the intervention rate and has the largest impact on the accuracy-efficiency tradeoff.
- **Avoid:** Applying primitives without lookahead validation. Key-similarity retrieval alone has a ~15% false positive rate — the 4-token probe step catches bad matches before they corrupt the reasoning trace.
- **Avoid:** Using a single global steering vector for all problems. The entire point of STIR is that different problems need different interventions at different points. Static vectors underperform by 3-5% compared to dynamic retrieval.

## Error Handling

- **Primitive library is empty after discovery:** Your rollouts lack sufficient variance. Increase K (number of rollouts), raise the sampling temperature, or use a more diverse problem set. Also check that the minimum centroid gap threshold isn't too strict.
- **Over-intervention degrades accuracy:** The null-action threshold is too low, or anchor entries are missing. Verify that anchor entries constitute ~40-50% of the library. Raise `tau_null` incrementally by 0.05 until accuracy stabilizes.
- **Lookahead probes are too expensive:** Reduce `T_probe` from 4 to 2 tokens, or batch candidates more aggressively. On a budget, skip probing and rely on retrieval similarity alone (accept ~2% accuracy trade-off).
- **Cross-task transfer fails:** The source and target domains are too dissimilar. Primitives transfer well within a domain (AIME → AMC) but poorly across domains (math → code). Build domain-specific libraries.
- **Library quality degrades over time:** As the model or agent improves, old primitives become stale. Re-run discovery periodically on recent failure cases. Implement a quality decay mechanism that down-weights primitives that haven't been successfully applied recently.

## Limitations

- Requires multiple rollouts per training problem (K=8 recommended), which means 8x the compute during the offline discovery phase. Not suitable for one-shot learning scenarios.
- The embedding space must be meaningful — if your state representations don't capture reasoning-relevant features, centroid differences will be noise. Works best with transformer hidden states or high-quality code/text embeddings.
- Anchor gating assumes that success and failure states are geometrically separable. On problems where correct and incorrect reasoning paths are nearly identical until the final step, the framework provides little benefit.
- Library size is bounded by retrieval efficiency. Beyond ~512 primitives, cosine similarity retrieval becomes noisy. For very diverse problem spaces, consider hierarchical or domain-partitioned libraries.
- The technique optimizes for problems where the model *can* solve the task some fraction of the time. If baseline success rate is near 0%, there are no successful rollouts to mine. STIR amplifies existing capability; it does not create new capability from scratch.

## Reference

Shi, Z., Zhu, Y., Shi, J., Zhang, X., & Wang, L. (2026). *Internalizing LLM Reasoning via Discovery and Replay of Latent Actions.* arXiv:2602.04925v1. [https://arxiv.org/abs/2602.04925v1](https://arxiv.org/abs/2602.04925v1)

Key sections to study: Algorithm 1 (Offline Construction) for the discovery and curation pipeline, Algorithm 2 (Online Intervention) for the retrieve-probe-inject loop, and Section 4.3 for cross-task transfer results showing that reasoning primitives generalize across related problem distributions.