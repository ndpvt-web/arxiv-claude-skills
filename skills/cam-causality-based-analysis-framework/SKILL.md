---
name: "cam-causality-based-analysis-framework"
description: "Analyze and optimize multi-agent code generation pipelines using causality-based importance ranking of intermediate features. Identifies which pipeline stages matter most, enables targeted failure repair, token-efficient pruning, and hybrid LLM backend assignment. Triggers: 'analyze my multi-agent pipeline', 'optimize agent code generation', 'which pipeline stages matter most', 'reduce token usage in my agent system', 'fix failing multi-agent code generation', 'assign LLMs to pipeline stages'"
---

# CAM: Causality-Based Analysis for Multi-Agent Code Generation Systems

This skill enables Claude to apply the CAM framework to analyze, diagnose, and optimize multi-agent code generation (MACGS) pipelines. CAM treats each intermediate output of a multi-agent pipeline (requirements, design docs, data structures, etc.) as a causal feature and quantifies its contribution to final code correctness using counterfactual error injection and a Feature Responsibility (FR) metric. This allows targeted failure repair (73.3% success rate on top-3 features), aggressive token pruning (up to 66.8% reduction), and hybrid LLM backend assignment (up to 7.2% Pass@1 gain).

## When to Use

- When a user has a multi-agent code generation pipeline (e.g., MetaGPT, ChatDev, Self-Collaboration, PairCoder) and wants to know which stages are bottlenecks
- When multi-agent code generation is failing and the user needs to diagnose which intermediate output is causing the failure
- When the user wants to reduce prompt/token costs in a multi-agent pipeline without hurting code quality
- When the user wants to assign different LLM backends to different pipeline stages based on each model's strengths
- When the user asks "why is my agent pipeline producing wrong code" or "which agent matters most"
- When designing a new MACGS pipeline and deciding which intermediate artifacts to include or skip

## Key Technique

**Counterfactual Feature Importance via Error Injection.** CAM works by injecting realistic errors into each intermediate output of a MACGS pipeline and measuring whether the final code breaks. Unlike random perturbation, CAM uses an LLM to generate semantically coherent but incorrect variants of each feature — plausible mistakes that mirror real failures. For each problem, CAM discovers *minimal* feature combinations whose simultaneous corruption causes failure, then aggregates these across problems into a Feature Responsibility (FR) score.

**Feature Responsibility (FR) Metric.** The FR score is computed as: `FR(f_i) = sum over problems p, sum over important sets S containing f_i, of (1/|S|)^2`. The squared inverse of set size amplifies features that cause failure on their own (|S|=1) versus those that only matter in combination. This distinguishes *independently critical* features (like `Data_Struct` and `Req_Stat`, which rank #1 across all configurations) from *context-dependent* features (like `Program_Lang`, which matters in 78.8% of cases only when corrupted alongside other features).

**Three Practical Applications.** (1) **Failure Repair**: When code generation fails, inspect and fix the top-3 FR-ranked features first — this achieves 73.3% repair success versus 27-41% for baselines. (2) **Feature Pruning**: Remove the lowest-FR features from the pipeline to cut tokens by 6-67% with negligible quality loss. (3) **Hybrid Backend**: Compare FR distributions across LLMs and assign each pipeline stage to the model that is strongest on that feature category — yields up to 7.2% Pass@1 improvement.

## Step-by-Step Workflow

1. **Map the pipeline's intermediate features.** Catalog every intermediate output your MACGS produces. Classify each into one of four categories:
   - **Specification**: task statement (`Req_Stat`), programming language (`Program_Lang`), documentation language (`Language`)
   - **Analysis**: requirement analysis (`Req_Anal`), competitive analysis (`Compet_Anal`), logic breakdown (`Logic_Anal`), requirement pool (`Req_Pool`)
   - **Design**: implementation approach (`Implement`), data structures (`Data_Struct`)
   - **Dependency**: required packages (`Req_Pack`), file listing (`File_List`), shared knowledge (`Share_Know`)

2. **Select a benchmark problem set.** Choose 50-164 problems from a relevant benchmark (HumanEval, MBPP, CodeContest, or your own test suite). These must have verifiable correctness criteria (test cases).

3. **Generate baseline outputs.** Run the pipeline end-to-end on the problem set, recording every intermediate feature and the final Pass@1 rate. Store these as ground-truth feature values.

4. **Inject counterfactual errors per feature.** For each feature `f_i` and each problem, prompt an LLM to produce a realistic but incorrect variant of `f_i` that is superficially plausible. Validate that the corrupted variant differs meaningfully (sentence-transformer cosine similarity < 0.5 with original).

5. **Discover minimal important feature sets.** For each problem, test single-feature corruptions first. If corruption of `f_i` alone causes failure, record `{f_i}` as an important set. For surviving features, test combinations of increasing length (2 up to `L_max=5`), using an influence-set heuristic to prioritize combinations likely to cause failure. Cap total executions at N=100 per problem with early stopping patience k=10.

6. **Compute Feature Responsibility scores.** Aggregate across all problems using `FR(f_i) = sum_p sum_{S in S_p, f_i in S} (1/|S|)^2`. Rank features by FR. The top features are your optimization targets.

7. **Apply failure repair (if diagnosing failures).** For each failing problem, inspect the top-3 FR-ranked features in order. Check each for semantic clarity, completeness, and consistency with the problem statement. Augment or correct the feature, then re-run the pipeline.

8. **Apply feature pruning (if reducing cost).** Remove the bottom-N FR-ranked features from the pipeline. Start conservatively (N=2) and measure Pass@1 delta. Increase N until quality degrades beyond your tolerance. Typical sweet spot: pruning 2-4 features for 6-34% token savings with no quality loss.

9. **Apply hybrid backend assignment (if using multiple LLMs).** Compare FR distributions across candidate LLMs. Assign each pipeline stage to the model with the lowest FR (fewest failures) on that feature category. For example, if DeepSeek excels at Design features and GPT-4o-mini at Specification features, route accordingly.

10. **Validate and iterate.** Re-run the full benchmark with your changes (repaired features, pruned pipeline, or hybrid backend). Compare Pass@1 against baseline. Iterate by adjusting the FR threshold or backend assignments.

## Concrete Examples

**Example 1: Diagnosing a Failing MetaGPT Pipeline**

User: "My MetaGPT pipeline is failing on 40% of HumanEval problems. Which intermediate step should I fix first?"

Approach:
1. Catalog MetaGPT's 12 intermediate features across Specification, Analysis, Design, and Dependency categories
2. Run CAM error injection on the 66 failing problems (inject realistic errors into each feature independently, then in combinations)
3. Compute FR scores — expect `Data_Struct` and `Req_Stat` to rank highest (consistent with paper findings across all configurations)
4. Inspect `Data_Struct` outputs on failing problems for incorrect structure choices (e.g., using a list where a hashmap is needed)
5. Augment the Data_Struct generation prompt with explicit guidance on structure selection rationale

Output:
```
Feature Responsibility Rankings (top 5):
1. Data_Struct (FR=14.2)  — Design category
2. Req_Stat   (FR=13.8)  — Specification category
3. Implement  (FR=9.1)   — Design category
4. Language   (FR=7.4)   — Specification category
5. Req_Anal   (FR=5.6)   — Analysis category

Recommendation: Focus repair on Data_Struct and Req_Stat first.
These two features appear in the important set for 31 of 66 failing problems.
Repairing top-3 features should fix ~73% of failures based on CAM benchmarks.
```

**Example 2: Reducing Token Costs in a Code Generation Pipeline**

User: "My multi-agent pipeline uses 12 intermediate features and costs too much. Which features can I safely remove?"

Approach:
1. Run CAM analysis to get FR scores for all 12 features
2. Identify the bottom-ranked features (typically `Compet_Anal`, `Share_Know`, `File_List`)
3. Remove bottom-2 features first, measure Pass@1 delta
4. Incrementally remove more if quality holds

Output:
```
Pruning Analysis:
- Remove bottom 2 (Compet_Anal, Share_Know): -6.4% tokens, +0.2% Pass@1 (slight improvement)
- Remove bottom 4 (+ File_List, Req_Pool):   -33.6% tokens, -0.8% Pass@1 (negligible loss)
- Remove bottom 6 (+ Logic_Anal, Req_Pack):  -51.2% tokens, -3.1% Pass@1 (moderate loss)
- Remove bottom 8:                            -66.8% tokens, -8.4% Pass@1 (significant loss)

Recommended: Prune bottom 4 features for 33.6% token savings with negligible quality impact.
```

**Example 3: Hybrid LLM Backend Assignment**

User: "I have access to GPT-4o-mini and DeepSeek. How should I split my pipeline between them?"

Approach:
1. Run CAM independently with each LLM as the full backend
2. Compare per-category FR distributions — lower FR means fewer failures on that category
3. Assign each category to the model with lower FR on those features
4. Wire the pipeline to route Specification/Analysis prompts to one model and Design/Dependency prompts to the other

Output:
```
Per-Category FR Comparison (lower = better):
                  GPT-4o-mini   DeepSeek
Specification:      8.2          11.4      → GPT-4o-mini
Analysis:           6.9           7.1      → GPT-4o-mini (marginal)
Design:            12.3           6.8      → DeepSeek
Dependency:         5.1           4.9      → DeepSeek (marginal)

Hybrid Assignment:
- Specification + Analysis stages → GPT-4o-mini
- Design + Dependency stages     → DeepSeek

Expected improvement: ~5-7% Pass@1 over uniform single-backend.
```

## Best Practices

- **Do:** Always start with single-feature error injection before testing combinations. Single-feature failures are the strongest causal signals and cheapest to compute.
- **Do:** Use LLM-generated counterfactual errors rather than random deletion or string mutation. Realistic errors produce reliable importance rankings (99% contextual appropriateness in paper validation).
- **Do:** Validate FR rankings against a held-out problem set before making pipeline changes. The paper achieved Kendall correlation of 0.76-0.91 with manual annotations.
- **Do:** Check for context-dependent features. A feature with low individual FR but high combinatorial FR (like `Program_Lang`) indicates cross-feature consistency issues — fix the interaction, not the feature in isolation.
- **Avoid:** Pruning more than 4 features at once without measuring intermediate quality. Aggressive pruning (6+ features) consistently degraded performance in the paper.
- **Avoid:** Assuming FR rankings generalize across all LLM backends. Rankings shift meaningfully between models — always recompute FR when switching backends.

## Error Handling

- **Error injection produces no behavioral change**: The similarity threshold (theta=0.5) may be too loose. Tighten it to 0.3 to ensure corrupted features differ sufficiently from originals. If the LLM generates variants too close to the original, add explicit instructions like "introduce a subtle but meaningful error."
- **Too many executions needed**: With 12 features and combinations up to length 5, the search space explodes. Use the influence-set heuristic from the paper: after injecting errors into set S, measure which other features `f_j` have their outputs change (similarity < theta). Prioritize combinations containing features with large influence sets.
- **FR scores are flat across features**: This indicates the pipeline's features are either all equally important or the error injection is not strong enough. Verify by checking that corrupted features actually differ from originals. Also try a harder benchmark — easy problems may tolerate any corruption.
- **Hybrid backend shows no improvement**: The candidate LLMs may have similar capability profiles. Hybrid assignment works best when models have complementary strengths (e.g., one excels at algorithmic design, another at specification parsing).

## Limitations

- CAM requires repeated pipeline executions (up to N=100 per problem) with error injection, making it computationally expensive for large problem sets or slow pipelines. Budget 50-100x your normal pipeline cost for a full analysis.
- The framework assumes intermediate features are identifiable and separable. Pipelines with tightly coupled or streaming architectures where outputs blend together are harder to analyze.
- FR rankings are specific to the problem distribution, LLM backend, and pipeline architecture. Rankings computed on HumanEval may not transfer to production workloads with different complexity profiles.
- The method requires test cases or other automated correctness checks. It cannot be applied to open-ended generation tasks where correctness is subjective.
- Context-dependent features (those important only in combination) are harder to detect and require combinatorial testing, which scales poorly beyond 5-feature combinations.

## Reference

**Paper:** [CAM: A Causality-based Analysis Framework for Multi-Agent Code Generation Systems](https://arxiv.org/abs/2602.02138v2) — Lyu et al., 2026. Look for Algorithm 1 (computing important feature sets), Equation 4 (FR metric), Tables 3-6 (importance rankings across configurations), and Section 5 (failure repair and pruning applications).