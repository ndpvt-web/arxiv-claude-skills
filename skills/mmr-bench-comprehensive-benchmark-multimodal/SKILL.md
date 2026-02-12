---
name: "mmr-bench-comprehensive-benchmark-multimodal"
description: "Build cost-aware multimodal LLM routing systems that select the best model per query based on input signals, budget constraints, and task type. Use when the user says 'route queries to different models', 'select the cheapest model that works', 'build a model router', 'multimodal model selection', 'cost-optimize LLM inference', or 'Pareto-optimal model dispatch'."
---

# Multimodal LLM Query Routing with Budget-Aware Model Selection

This skill enables Claude to design and implement **query-level model routing systems** that direct each incoming request to the most cost-effective multimodal LLM from a candidate pool. Based on the MMR-Bench framework, the core insight is that no single model dominates all tasks -- a router that extracts text and image signals from each query, predicts per-model utility, and selects under a cost constraint can match the strongest single model's accuracy at roughly 33% of its inference cost. This skill covers the full pipeline: feature extraction via CLIP embeddings, adaptive modality fusion, router training (from k-means to MLP matrix factorization), cost-aware selection via a tunable lambda objective, and Pareto frontier evaluation.

## When to Use

- When the user wants to build a system that dispatches queries across multiple LLM APIs (e.g., GPT-5, Gemini, Claude, Qwen) based on query difficulty and modality
- When designing cost-optimization infrastructure for multimodal inference workloads that span OCR, VQA, and math reasoning
- When the user asks to implement a model router, gateway, or proxy that picks the cheapest model capable of answering each query correctly
- When evaluating routing policies with budget-aware metrics (nAUC, Quality-Neutral Cost, Pareto peak score)
- When building an adaptive inference layer that handles both image+text and text-only queries without separate routing logic
- When the user needs to benchmark or compare routing strategies against oracle and random baselines

## Key Technique

**Query-level routing** treats model selection as an optimization problem solved independently per input. For each query `i` and candidate model `j`, the router predicts a utility score `u_hat(i,j)` (expected accuracy) and a cost `c_hat(i,j)` (normalized API price). The selection rule is: `j*(i; lambda) = argmin_j (1 - u_hat(i,j) + lambda * c_hat(i,j))`, where `lambda` is a budget-sensitivity knob swept from 0 (accuracy-only) to large values (cost-only) to trace the full Pareto frontier.

**Adaptive modality fusion** is what makes multimodal routing superior to text-only approaches. Both the text prompt and the image (if present) are encoded with frozen CLIP encoders into L2-normalized embeddings. A confidence estimator combines prototype-similarity and norm-based sigmoid scores to produce per-modality weights via softmax (temperature eta=5.0). The fused representation includes three interaction terms -- weighted sum, element-wise product, and absolute difference -- which teach the router to reason about agreement and mismatch between text and image channels. This fusion degrades gracefully: when no image is present, the text channel dominates automatically, enabling zero-shot transfer to text-only benchmarks.

**Router architectures** range from non-parametric (KMeansRouter assigning queries to cluster centroids with precomputed per-cluster utility; KNNRouter averaging neighbors) to learned models (LinearRouter, 2-layer MLPRouter) to matrix-factorization variants (MLPMFRouter projecting queries into a shared latent space with model-specific weight vectors). The MF variants are particularly effective because they learn a low-rank structure over the query-model utility matrix, capturing that "models good at OCR" share latent factors distinct from "models good at spatial reasoning."

## Step-by-Step Workflow

1. **Define the candidate model pool with cost metadata.** List every model available for routing (e.g., GPT-5 at $10/M tokens, Gemini 2.5 Flash at $2/M, Qwen2.5-VL-7B at $0.07/M). Normalize costs: `c_j = raw_cost_j / max(all_raw_costs)`. Store as a config dict mapping model name to normalized cost.

2. **Build the evaluation outcome table.** For each (query, model) pair in your training set, record the binary or continuous utility score (did the model answer correctly?). This offline table eliminates the need to re-run models during router training. Schema: `{query_id, model_id, utility, cost}`.

3. **Extract query embeddings with CLIP.** Use a frozen CLIP model (e.g., `openai/clip-vit-large-patch14`) to encode the text prompt and image separately. L2-normalize both embeddings. For text-only queries, set the image embedding to a zero vector.

4. **Implement adaptive modality fusion.** Compute confidence scores per modality: (a) cosine similarity to a modality prototype (mean embedding of training set), shifted to [0,1]; (b) norm-based sigmoid `sigma((norm - mean_norm) / std_norm)`; (c) blend both with equal weight, clip to [0,1]. Apply softmax with temperature 5.0 over [text_conf, image_conf] to get weights. Construct the fused vector as `concat(weighted_sum, elementwise_product, abs_difference)`, then L2-normalize.

5. **Train the router on the outcome table.** Choose an architecture based on pool size and data volume:
   - **< 1K training queries**: KMeansRouter (k=num_models * 5 clusters, assign centroids, precompute cluster-level utilities)
   - **1K-10K queries**: MLPRouter (2-layer MLP, hidden_dim=256, predict per-model utility from fused embedding)
   - **10K+ queries or 10+ models**: MLPMFRouter (project fused embedding to latent dim=64, learn model-specific vectors, dot-product for utility)

   Loss: MSE between predicted and actual utility. Train with AdamW, lr=1e-3, for 50 epochs.

6. **Implement the cost-aware selection function.** At inference, compute `score_j = (1 - u_hat_j) + lambda * c_hat_j` for each candidate model. Select `argmin(score)`. Sweep `lambda` over `[0, 0.1, 0.5, 1.0, 2.0, 5.0, 10.0]` to generate the Pareto frontier during evaluation.

7. **Evaluate with budget-aware metrics.** Compute three metrics from the Pareto curve: (a) **nAUC** -- area under the performance-vs-cost curve normalized by the achievable range; (b) **Peak Score (Ps)** -- maximum accuracy on the frontier; (c) **Quality-Neutral Cost (QNC)** -- the minimum relative cost at which the router matches the best single model's accuracy. QNC < 1.0 means cost savings; QNC ~ 0.33 is the target from the paper.

8. **Add oracle and random baselines.** OracleRouter always picks the model with highest actual utility (upper bound). RandomRouter selects uniformly (lower bound). Both are essential for interpreting nAUC and validating the router adds real value.

9. **Test zero-shot generalization.** Evaluate the trained router on held-out datasets and text-only benchmarks without retraining. The adaptive fusion naturally degrades the image channel when absent, so the same router works across modalities.

10. **Deploy as an inference gateway.** Wrap the router in an API layer: accept a query (text + optional image), run CLIP encoding + fusion + selection, then forward the query to the chosen model's API. Log the routing decision, predicted utility, actual utility, and cost for continuous improvement.

## Concrete Examples

**Example 1: Building a cost-optimized vision-language API gateway**

```
User: I have access to GPT-5, Gemini 2.5 Flash, and Qwen2.5-VL-7B via API.
I want to route incoming OCR and VQA queries to minimize cost while keeping
accuracy above 90%. Help me build the router.

Approach:
1. Define cost table:
   models = {
       "gpt5": {"api": "openai", "cost_per_1m": 10.00, "norm_cost": 1.0},
       "gemini_flash": {"api": "google", "cost_per_1m": 2.00, "norm_cost": 0.2},
       "qwen_7b": {"api": "together", "cost_per_1m": 0.07, "norm_cost": 0.007},
   }

2. Collect outcome table: Run all 3 models on 2,000 labeled OCR + VQA queries.
   Record binary correctness per (query, model) pair.

3. Extract CLIP embeddings for each query (text + image).

4. Train MLPRouter with adaptive fusion on the outcome table.

5. Sweep lambda to find the value where accuracy >= 90% at minimum cost.
   Typical result: lambda=1.5 routes ~60% of easy OCR to qwen_7b,
   ~30% of moderate VQA to gemini_flash, ~10% of hard reasoning to gpt5.

Output: A FastAPI service that accepts multimodal queries and returns
the selected model's response, with routing metadata in headers:
  X-Router-Model: gemini_flash
  X-Router-Predicted-Utility: 0.94
  X-Router-Cost-Savings: 0.78
```

**Example 2: Evaluating a routing policy against baselines**

```
User: I built a simple rule-based router that sends OCR to a small model
and everything else to GPT-5. How do I know if it's any good?

Approach:
1. Build an outcome table on a held-out test set of 1,000 queries across
   OCR, VQA, and math reasoning tasks.

2. Compute the user's router decisions and look up actual utilities/costs
   from the outcome table.

3. Compute the three MMR-Bench metrics:
   - nAUC: Plot accuracy vs. cumulative cost, compute normalized AUC
   - Peak Score: Best accuracy the router achieves
   - QNC: Cost fraction at which it matches the best single model

4. Compare against baselines:
   - OracleRouter (always picks correct model): nAUC=0.92, Ps=0.97, QNC=0.28
   - RandomRouter: nAUC=0.41, Ps=0.72, QNC=1.0
   - Rule-based router: nAUC=0.63, Ps=0.89, QNC=0.71
   - Trained MLPMFRouter: nAUC=0.81, Ps=0.95, QNC=0.34

Output: The rule-based router saves some cost (QNC=0.71) but leaves
significant value on the table. A trained router nearly triples the
cost savings (QNC=0.34) while improving peak accuracy by 6 points.
```

**Example 3: Adding a new model to an existing routing pool**

```
User: We just got access to Claude 3.7 Sonnet. How do I add it to our
existing 5-model router without retraining from scratch?

Approach:
1. Run Claude 3.7 Sonnet on the existing training queries to extend
   the outcome table with a new column.

2. Add its cost entry: {"claude_sonnet": {"cost_per_1m": 15.00, "norm_cost": 1.0}}
   (re-normalize all costs if the new model changes the max).

3. For MLPMFRouter: freeze the query encoder and existing model vectors.
   Initialize a new model vector w_claude and bias b_claude. Fine-tune
   only these parameters for 10 epochs on the extended outcome table.

4. For KMeansRouter: recompute cluster-level utilities to include the
   new model column. No retraining needed.

5. Re-evaluate: sweep lambda and check if Claude 3.7 Sonnet appears
   on the Pareto frontier. If it dominates another model at similar cost,
   consider removing the dominated model from the pool.

Output: Updated router config with 6 models. Claude 3.7 Sonnet is selected
for ~15% of hard multimodal reasoning queries where it outperforms GPT-5,
improving Ps from 0.95 to 0.96 with minimal cost increase.
```

## Best Practices

- **Do:** Normalize all costs to [0, 1] by dividing by the maximum cost in the pool. This ensures lambda has consistent meaning across different model sets.
- **Do:** Always include at least one cheap model (< $0.10/M tokens) and one strong model (> $5/M tokens) in the candidate pool. Routing only helps when models differ meaningfully in both capability and cost.
- **Do:** Use the adaptive fusion mechanism even if most queries are text-only. The confidence weighting gracefully down-weights the image channel, and the interaction features still capture useful signal from text embeddings alone.
- **Do:** Evaluate with all three metrics (nAUC, Ps, QNC) -- a router can look good on one metric while failing another (e.g., high peak score but poor cost efficiency).
- **Avoid:** Training the router on the same data used to build the outcome table without a train/test split. The outcome table is ground truth; the router must generalize to unseen queries.
- **Avoid:** Using raw API latency as the cost signal. Latency varies with load and is not reproducible. Use published pricing per output token, which is stable and comparable.
- **Avoid:** Routing with text-only features when images are available. The paper shows text-only routing underperforms significantly on vision-governed tasks like OCR, chart reading, and spatial reasoning -- multimodal fusion is not optional for multimodal workloads.

## Error Handling

- **Unbalanced outcome table (some models missing on some queries):** Impute missing utilities with the model's mean score on that task category. Flag imputed entries and exclude them from evaluation metrics.
- **CLIP encoding fails on unusual image formats:** Preprocess all images to RGB PNG/JPEG before encoding. For PDF pages or multi-frame inputs, extract the first frame or render at 224x224.
- **Router always selects the same model:** This indicates lambda is too extreme (all-accuracy or all-cost) or the training set lacks diversity. Verify the outcome table has variance across models, and sweep lambda across a wider range.
- **New model dominates the cost-accuracy frontier completely:** If one model is both cheapest and most accurate on all queries, routing adds no value. This is the correct outcome -- remove routing overhead and use that model directly.
- **Zero-shot transfer to text-only fails:** Verify the fusion confidence scores. If image confidence is non-zero on text-only inputs (due to noise), add an explicit check: set image embedding to zero vector when no image is provided.

## Limitations

- **Requires an offline outcome table:** You must run every candidate model on your training queries to build ground truth. For K=10 models and N=10,000 queries, that is 100,000 inference calls. This upfront cost is significant but amortized over production traffic.
- **Static cost model:** The routing decision uses fixed per-token pricing. It does not account for variable-length outputs, rate limits, or dynamic pricing. For streaming or real-time workloads, actual cost may diverge from predicted cost.
- **CLIP embeddings may not capture task difficulty:** CLIP is trained for image-text alignment, not difficulty estimation. Queries that are semantically similar but differ in reasoning complexity may get similar embeddings. Fine-tuning the encoder or adding hand-crafted difficulty features can help.
- **Assumes independent queries:** The router selects per-query without considering batch-level constraints (e.g., "spend at most $X total for this batch"). Batch-level budget optimization requires a different formulation.
- **Model pool changes require partial retraining:** Adding or removing a model invalidates part of the outcome table and router parameters. The MF architecture mitigates this by isolating model-specific vectors, but full retraining may still be needed for large pool changes.

## Reference

**MMR-Bench: A Comprehensive Benchmark for Multimodal LLM Routing** (Ma, Lai, Ye, 2026). arXiv:2601.17814. Look for: Section 3 (routing formulation and cost model), Section 4 (adaptive fusion mechanism), Section D.2 (confidence estimation and interaction features), Table 2 (main routing results showing QNC ~0.33), and Table 5 (zero-shot transfer to text-only benchmarks).