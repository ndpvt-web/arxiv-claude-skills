---
name: "memcast-memory-driven-time-series"
description: "Build memory-augmented time series forecasting systems using hierarchical experience storage (historical patterns, reasoning wisdom, general laws) with LLM-driven inference and reflective iteration. Trigger phrases: 'forecast time series with memory', 'experience-driven prediction', 'memcast forecasting', 'hierarchical memory for time series', 'LLM time series with retrieval', 'memory-augmented forecasting pipeline'"
---

# MemCast: Memory-Driven Time Series Forecasting with Experience-Conditioned Reasoning

This skill enables Claude to design and implement time series forecasting systems that accumulate structured experience from training data into a three-layer hierarchical memory — historical patterns, reasoning wisdom, and general laws — then use that memory to condition LLM-based inference at prediction time. The approach, from the MemCast paper, treats forecasting not as a single forward pass but as a retrieve-reason-reflect loop where past experience guides trajectory selection and domain constraints enforce correctness.

## When to Use

- When the user wants to build an LLM-based time series forecasting pipeline that improves over time by learning from its own prediction history
- When the user asks to add retrieval-augmented generation (RAG) to a time series prediction system
- When the user needs to forecast volatile series (energy prices, stock movements, weather) where pure statistical models struggle with regime changes
- When the user wants to implement a reflect-and-revise loop for numeric predictions, where domain constraints correct LLM outputs
- When the user asks how to store and reuse reasoning trajectories (chain-of-thought paths) that led to good vs. bad predictions
- When the user needs continual learning for forecasting without retraining — updating confidence scores on memory entries as new data arrives

## Key Technique

MemCast reformulates time series forecasting as an **experience-conditioned reasoning task**. Instead of training a model end-to-end, it builds a structured memory from training data and retrieves relevant experience at inference time to guide an LLM forecaster. The memory has three layers: (1) **Historical Pattern Memory** stores natural-language summaries of past input-output pairs (trend direction, volatility level, peak values), enabling semantic retrieval of similar situations. (2) **Reasoning Wisdom Memory** stores distilled reasoning trajectories partitioned into successes and failures, using a composite similarity metric (alpha * cosine_embedding_similarity + (1-alpha) * DTW_structural_similarity) with deduplication thresholds to keep the store compact. (3) **General Laws Memory** stores induced domain rules (e.g., "prices cannot be negative," "output must be continuous with last observed value") extracted from statistical features of training data.

During inference, MemCast executes a three-phase loop: **Retrieve** top-k similar historical patterns to form context, **Reason** by sampling M independent LLM trajectories and scoring them against reasoning wisdom for semantic consistency, then **Reflect** by checking the best prediction against general law constraints and re-prompting the LLM with violation details if any rule fails. This loop continues until constraints are satisfied or a retry limit is reached.

The system also supports **dynamic confidence adaptation** — each memory entry carries a mutable confidence score (initialized to zero) that increases when the entry contributes to a prediction that beats a moving-average baseline. At retrieval time, confidence scores weight the similarity ranking, so entries that prove useful in practice rise to prominence while stale entries decay in influence. Critically, confidence updates use only the prediction vs. baseline comparison, never ground-truth test labels, preventing test-set leakage.

## Step-by-Step Workflow

1. **Prepare the training data**: Load time series with lookback windows (input) and forecast horizons (output). For each training sample, compute basic statistical features: trend slope, volatility (rolling std), min/max values, periodicity via autocorrelation, and any exogenous covariates.

2. **Build Historical Pattern Memory**: For each training sample, generate a natural-language summary of the input window and the ground-truth output. Use an LLM to produce structured text like: "Input shows a rising trend with moderate volatility (std=2.3), peaking at 45.2 on day 3. Output continues the rise, reaching 51.8 before a reversal on day 6." Store as key-value pairs: (input_embedding, output_summary).

3. **Build Reasoning Wisdom Memory**: Run the LLM forecaster on training samples, collecting full chain-of-thought trajectories. Partition into successes (MAE < threshold tau) and failures (MAE >= tau). For each set, deduplicate using composite similarity S = alpha * cosine_sim(embeddings) + (1-alpha) * (1 - normalized_DTW_distance). If S > 0.95, replace the older entry. If 0.8 < S <= 0.95, merge entries via LLM summarization. If S <= 0.8, keep both. Store as positive wisdom and negative wisdom collections.

4. **Build General Laws Memory**: Extract statistical features across the full training set (global min/max bounds, continuity expectations, seasonal patterns, covariate correlations). Use the LLM to induce general rules from these features, e.g., "Electricity prices in this market are bounded [−50, 500] EUR/MWh" or "Streamflow values are strictly non-negative and exhibit strong 365-day seasonality." Store as a list of checkable constraints.

5. **Implement the retrieval module**: Given a new input window at inference time, compute its embedding and retrieve the top-k (k=3 works well) most similar historical patterns using the composite similarity metric. Format retrieved patterns as context text for the LLM prompt.

6. **Implement wisdom-guided trajectory sampling**: Prompt the LLM with the input data, retrieved patterns, and statistical context. Sample M=4 independent reasoning trajectories at temperature=0.6. Score each trajectory against positive wisdom (higher semantic similarity = better) and negative wisdom (lower similarity = better). Select the trajectory with the highest combined score.

7. **Implement law-based reflective iteration**: Check the selected prediction against each general law constraint. If any constraint fails (e.g., predicted value < 0 when non-negativity is required), construct a corrective prompt: "Your prediction violates constraint: [rule]. Specifically, value at step t=5 is -3.2 but must be >= 0. Revise your forecast." Re-run the LLM with this feedback. Repeat up to 3 iterations.

8. **Implement dynamic confidence adaptation**: After each inference, compare the LLM prediction loss against a moving-average baseline prediction. If the LLM wins, increment confidence scores for all memory entries that were retrieved for this prediction by a small delta (e.g., +0.1). Do not decrement on failure — only upward updates to maintain stability. At retrieval time, rank by: combined_score = similarity * (1 + confidence).

9. **Assemble the end-to-end pipeline**: Wire together data loading, memory construction (offline, once), and the retrieve-reason-reflect inference loop (online, per query). Expose configuration for: top-k retrieval count, number of trajectory samples M, sampling temperature, reflection retry limit, confidence update delta, and similarity alpha.

10. **Evaluate and iterate**: Measure MSE and MAE on held-out test data. Compare against baselines (moving average, ARIMA, vanilla LLM prompting without memory). Inspect which memory entries have highest confidence scores to validate that the system is learning meaningful patterns.

## Concrete Examples

**Example 1: Energy Price Forecasting Pipeline**

User: "I have hourly electricity price data for Nord Pool with exogenous features (load forecast, wind generation, gas prices). Build a forecasting system that uses past prediction experience to improve over time."

Approach:
1. Load the Nord Pool CSV with columns: timestamp, price, load_forecast, wind_gen, gas_price
2. Create sliding windows: 168h lookback (1 week), 24h forecast horizon
3. Build historical pattern memory by summarizing each training window: "Week shows high volatility (std=12.4 EUR), with a demand-driven spike to 89 EUR on Wednesday evening. Following 24h saw prices normalize to 45-55 EUR range as wind generation increased."
4. Run LLM forecaster on training set, collect 4 trajectories per sample at temp=0.6, partition into success/failure wisdom at MAE threshold of 5.0 EUR
5. Induce general laws: "Prices bounded [-50, 500]", "Adjacent hour price changes rarely exceed 40 EUR", "High wind generation correlates with lower prices"
6. At inference: retrieve 3 similar weeks, sample 4 trajectories, score against wisdom, reflect against constraints, output 24h forecast

Output:
```python
# Simplified pipeline structure
class MemCastForecaster:
    def __init__(self, llm_client, embedding_model, config):
        self.historical_memory = HistoricalPatternStore(embedding_model)
        self.wisdom_memory = ReasoningWisdomStore(embedding_model, tau=5.0)
        self.laws_memory = GeneralLawsStore()
        self.confidence = ConfidenceTracker(delta=0.1)
        self.config = config  # top_k=3, M=4, temp=0.6, max_reflect=3

    def build_memory(self, train_data):
        for window, target in train_data:
            summary = self.llm_client.summarize(window, target)
            self.historical_memory.add(window, summary)
        # ... wisdom distillation and law induction

    def forecast(self, input_window):
        similar = self.historical_memory.retrieve(input_window, k=self.config.top_k)
        trajectories = [self.llm_client.reason(input_window, similar, temp=0.6)
                        for _ in range(self.config.M)]
        best = self.wisdom_memory.select_best(trajectories)
        for attempt in range(self.config.max_reflect):
            violations = self.laws_memory.check(best.prediction)
            if not violations:
                break
            best = self.llm_client.revise(best, violations)
        self.confidence.update(similar, best.prediction, input_window)
        return best.prediction
```

**Example 2: Retail Demand Forecasting with Reflection**

User: "My LLM forecaster sometimes predicts negative demand values for retail products. How do I add a constraint-checking layer that catches and fixes these?"

Approach:
1. Define general law constraints from domain knowledge: demand >= 0, demand <= max_historical * 1.5, daily demand should be continuous (no jumps > 3x previous day)
2. After the LLM generates a forecast, run each constraint as a Python check
3. On violation, construct a corrective prompt with the specific failure and re-query the LLM
4. Cap reflection at 3 iterations — if still failing, clamp values programmatically as fallback

Output:
```python
class LawBasedReflector:
    def __init__(self, laws, llm_client, max_iterations=3):
        self.laws = laws  # List of callable constraint checks
        self.llm = llm_client
        self.max_iter = max_iterations

    def reflect(self, prediction, context):
        for i in range(self.max_iter):
            violations = []
            for law in self.laws:
                result = law.check(prediction)
                if not result.passed:
                    violations.append(result.description)
            if not violations:
                return prediction
            prompt = (f"Your forecast violates these constraints:\n"
                      + "\n".join(f"- {v}" for v in violations)
                      + "\nRevise the forecast to satisfy all constraints.")
            prediction = self.llm.revise(prediction, context, prompt)
        # Fallback: programmatic clamping
        return self.clamp_to_constraints(prediction)

# Constraint definitions
laws = [
    BoundConstraint("demand >= 0", lambda y: all(v >= 0 for v in y)),
    BoundConstraint("demand <= historical max * 1.5",
                    lambda y: all(v <= max_hist * 1.5 for v in y)),
    ContinuityConstraint("no 3x jumps between adjacent steps", max_ratio=3.0),
]
```

**Example 3: Adding Confidence-Based Memory Evolution**

User: "I have a deployed forecasting system. How do I update which historical patterns are most trusted without retraining on test data?"

Approach:
1. Maintain a confidence score per memory entry, initialized to 0
2. After each live prediction, compare LLM forecast error against a simple moving-average baseline
3. If LLM beats baseline, increment confidence for all retrieved entries by +0.1
4. At retrieval time, re-rank by: score = similarity * (1 + confidence)
5. Never decrement — this avoids instability from noisy single predictions

Output:
```python
class ConfidenceTracker:
    def __init__(self, delta=0.1):
        self.scores = {}  # memory_id -> float
        self.delta = delta

    def update(self, retrieved_ids, llm_loss, baseline_loss):
        if llm_loss < baseline_loss:  # LLM beat moving average
            for mid in retrieved_ids:
                self.scores[mid] = self.scores.get(mid, 0.0) + self.delta

    def adjust_ranking(self, candidates):
        """Re-rank retrieval candidates by confidence-weighted similarity."""
        for c in candidates:
            c.final_score = c.similarity * (1.0 + self.scores.get(c.id, 0.0))
        return sorted(candidates, key=lambda c: c.final_score, reverse=True)
```

## Best Practices

- **Do:** Use composite similarity (semantic + structural via DTW) for retrieval — pure embedding cosine similarity misses temporal shape, while pure DTW misses semantic context
- **Do:** Set the wisdom partition threshold tau based on your domain's acceptable error — for energy prices, MAE < 5 EUR works well; for demand counts, use relative MAPE < 10%
- **Do:** Keep general laws as hard constraints with programmatic fallbacks — if 3 reflection iterations fail, clamp values rather than serving impossible predictions
- **Do:** Deduplicate wisdom entries aggressively (merge threshold 0.8-0.95) to keep retrieval fast and avoid redundant context that wastes LLM tokens
- **Avoid:** Updating confidence scores using ground-truth test labels — this leaks test distribution; compare only against a baseline forecast (moving average, naive persistence)
- **Avoid:** Using too many retrieval results (k > 5) or trajectory samples (M > 6) — this inflates LLM context and cost with diminishing returns; k=3, M=4 is the empirically validated sweet spot

## Error Handling

- **LLM returns non-numeric output**: Parse the LLM response with a structured extraction step. If parsing fails, retry with an explicit format instruction: "Return exactly N comma-separated float values." After 2 parse failures, fall back to the moving-average baseline.
- **No similar patterns found** (all similarity scores below threshold): Skip retrieval-augmented context and run the LLM with statistical features only. Log this event — it indicates the training set may not cover the current regime.
- **Reflection loop does not converge**: After max iterations, apply programmatic clamping (clip to bounds, enforce continuity via interpolation). Always serve a prediction rather than failing silently.
- **Memory grows too large**: Implement a pruning strategy — remove entries with zero confidence that haven't been retrieved in the last N inference cycles. Alternatively, set a hard cap and evict lowest-confidence entries.
- **Embedding model mismatch**: Ensure the same embedding model is used for memory construction and retrieval. Switching embedding models invalidates all stored similarity relationships and requires full memory rebuild.

## Limitations

- **LLM cost scales with trajectory sampling**: Sampling M=4 trajectories per prediction means 4x the LLM API calls. For high-frequency data (minute-level), this may be prohibitively expensive. Consider reducing M or batching predictions.
- **Memory construction is offline and slow**: Building all three memory layers requires running the LLM over the full training set multiple times. This is a one-time cost but can take hours for large datasets.
- **Domain-specific law induction requires human review**: The LLM-induced general laws may be incorrect or too loose. Always validate induced constraints against domain expertise before deploying.
- **Not suited for truly novel regimes**: If the test distribution is fundamentally unlike anything in training (e.g., a market structure change), historical pattern retrieval will return poor matches. The confidence adaptation helps but cannot overcome a complete distribution shift.
- **Requires an embedding model alongside the LLM**: The pipeline depends on a separate embedding model (e.g., sentence-transformers) for similarity computation, adding infrastructure complexity.

## Reference

**Paper:** [MemCast: Memory-Driven Time Series Forecasting with Experience-Conditioned Reasoning](https://arxiv.org/abs/2602.03164v1) — Tao et al., 2026. Focus on Section 3 for the hierarchical memory construction algorithm, Section 4 for the retrieve-reason-reflect inference loop, and Section 5 for ablation results showing each memory layer's contribution.

**Code:** [github.com/Xiaoyu-Tao/MemCast-TS](https://github.com/Xiaoyu-Tao/MemCast-TS) — Reference implementation with scripts for memory accumulation (`scripts/accumulation/`) and forecasting (`scripts/short_term/`, `scripts/long_term/`).