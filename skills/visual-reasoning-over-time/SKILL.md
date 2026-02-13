---
name: "visual-reasoning-over-time"
description: "Analyze time series data using the MAS4TS Analyzer-Reasoner-Executor multi-agent paradigm: convert series to plots, extract visual anchors via VLM reasoning, reconstruct trajectories in latent space, and route to task-specific tools. Use when: 'analyze this time series', 'forecast these values visually', 'detect anomalies in my sensor data', 'classify this temporal pattern', 'impute missing time series values', 'build a multi-agent time series pipeline'."
---

# Visual Reasoning over Time Series via Multi-Agent System

This skill teaches Claude to build time series analysis systems using the MAS4TS paradigm from Ruan & Liang (2026). Instead of feeding raw numeric sequences directly into models, the approach converts time series into plot images, uses a Vision-Language Model to extract sparse semantic anchors (critical turning points, regime shifts, trend indicators), then reconstructs full predictive trajectories in a continuous latent space conditioned on those anchors. Three specialized agents -- Analyzer, Reasoner, Executor -- coordinate through shared memory with gated communication, while an adaptive router selects task-specific tool chains. This unified framework handles forecasting, classification, anomaly detection, and imputation without task-specific architectures.

## When to Use

- When the user wants to forecast time series values and needs a structured multi-agent pipeline rather than a single monolithic model
- When the user asks to detect anomalies in sensor, infrastructure, or financial time series data
- When the user needs to classify temporal patterns (e.g., activity recognition, medical signal classification)
- When the user has time series with missing values and needs anchor-guided imputation
- When the user wants to combine visual inspection of plots with numeric analysis for explainable time series reasoning
- When the user is building an agent system that must handle multiple time series tasks through a single framework
- When the user wants to extract interpretable structural features (trend direction, periodicity, regime changes) from time series before modeling

## Key Technique

**The Analyzer-Reasoner-Executor Paradigm.** MAS4TS decomposes time series analysis into three agent roles. The Analyzer extracts statistical features (mean, variance, autocorrelation, spectral peaks) and renders task-aware plot images from the raw series. The Reasoner performs two-stage reasoning: first, visual reasoning via a VLM that examines the plot with structured prompts to extract *temporal anchors* -- sparse tuples of (critical_time_index, estimated_value, local_behavior) where local_behavior is one of falling/stable/rising. Second, it performs numeric reasoning by reconstructing full sequences in latent space using a latent ODE solver conditioned on those anchors. The Executor invokes task-specific tools, validates outputs against anchor-implied constraints, and applies corrections.

**Visual Anchors as Soft Boundary Regularization.** The core insight is that plotting a time series and examining it visually reveals structural patterns (trend breaks, seasonal peaks, anomalous spikes) that are difficult to extract from raw numbers alone. By having a VLM identify these as sparse anchors, the system creates semantically meaningful constraints for latent trajectory reconstruction. The anchor-conditioned ODE `dz(t)/dt = f(z(t), t | anchors, priors)` produces trajectories that are both mathematically smooth and structurally faithful to observed patterns. This is far more sample-efficient than end-to-end approaches.

**Gated Shared Memory and Adaptive Routing.** Agents communicate through a shared memory matrix, where each agent's contribution is gated by a learned confidence score (alpha in [0,1]). This prevents noisy or low-confidence outputs from corrupting shared state. An adaptive router then selects tool chains based on task type, extracted priors, and memory state, composing tools sequentially: `output = Tool_J(...(Tool_1(latent_representation)))`. This enables a single system to handle forecasting (patch-transformer tools), classification (temporal convolution tools), imputation (anchor-constrained completion), and anomaly detection (multi-scale scoring) through different tool compositions.

## Step-by-Step Workflow

1. **Ingest and normalize the time series data.** Load the raw time series into a structured format (pandas DataFrame or numpy array). Apply standard normalization (z-score or min-max) and record normalization parameters for later inversion. Identify metadata: sampling frequency, number of channels, series length, and any known seasonality periods.

2. **Compute statistical priors (Analyzer phase 1).** Extract a feature vector including: rolling mean and variance at multiple window sizes, dominant spectral frequencies via FFT, autocorrelation at standard lags (1, 7, 14, 30 steps), stationarity test results (ADF test p-value), and trend slope via linear regression. Store these as the structured prior dictionary `P`.

3. **Render task-aware time series plots (Analyzer phase 2).** Generate matplotlib plots of the series with task-specific annotations: for forecasting, mark the train/forecast split with a vertical dashed line; for anomaly detection, overlay rolling statistics bands; for imputation, highlight missing regions; for classification, show the full window with amplitude markers. Save plots as images at 512x384 resolution with clean axes and grid lines.

4. **Extract temporal anchors via VLM visual reasoning (Reasoner phase 1).** Send each plot image to a Vision-Language Model (GPT-4o, Claude with vision, or Gemini) with a structured prompt requesting extraction of 5-15 temporal anchors. Each anchor is a tuple: `(time_index: int, value_estimate: float, behavior: "rising"|"falling"|"stable")`. The prompt must include the statistical priors and the task type to focus the VLM's attention on task-relevant features.

5. **Reconstruct latent trajectories conditioned on anchors (Reasoner phase 2).** Embed the normalized series via patch embedding (patch size 16, stride 8). Initialize a latent state vector and solve a neural ODE forward in time, using the extracted anchors as soft constraints that bias the trajectory toward anchor values at anchor time indices. Fuse the ODE output with patch embeddings via cross-attention to produce a dense, semantically grounded latent representation.

6. **Route to task-specific tool chain (Router).** Based on task type and priors, select the appropriate tool composition: forecasting uses `[patch_embed -> self_attention -> projection_head]`; classification uses `[patch_embed -> temporal_conv -> pooling -> classifier]`; anomaly detection uses `[patch_embed -> multi_scale_scorer -> threshold]`; imputation uses `[patch_embed -> anchor_constrained_infill -> projection]`. Implement routing as a softmax over tool chain indices conditioned on task type, priors, and memory state.

7. **Execute the tool chain (Executor phase 1).** Run the selected tools sequentially on the latent representation. Each tool validates its input/output shapes and value ranges. For forecasting, decode latent states to future values and invert normalization. For classification, produce class logits. For anomaly detection, produce per-timestep anomaly scores. For imputation, produce completed series values.

8. **Validate and correct outputs (Executor phase 2).** Check outputs against anchor-implied constraints: forecasted values should be consistent with the trend direction of nearby anchors; anomaly scores should spike near anchor points marked as regime changes; imputed values should respect anchor value estimates. Apply soft constraint projection to bring outputs within plausible bounds. For classification, enforce schema consistency (valid class labels, proper probability normalization).

9. **Update shared memory and aggregate results.** Write the Executor's output and confidence score to shared memory. If running iteratively, allow the Analyzer to re-examine residuals. Produce the final output with both numeric results and an interpretability summary referencing the extracted anchors and their influence on the result.

10. **Return structured output with explanation.** Deliver results in a format appropriate to the task (forecast array, class label with confidence, anomaly mask with scores, or completed series) alongside a natural-language explanation that references the visual anchors: "The model identified a trend reversal at index 142 (rising -> falling) which anchored the forecast decline."

## Concrete Examples

**Example 1: Forecasting energy consumption**

```
User: I have hourly electricity consumption data for the past 30 days.
Forecast the next 48 hours.

Approach:
1. Load the 720-point series, apply z-score normalization
2. Compute priors: 24h dominant spectral peak, strong autocorrelation
   at lag 24, slight upward trend (slope=0.003/hour)
3. Plot the series with a vertical line at t=720 marking forecast start,
   overlay 24h rolling mean
4. Send plot to VLM with prompt:
   "Identify 8-12 critical turning points in this hourly energy series.
    For each, report (hour_index, consumption_estimate, trend_direction).
    The series has 24h seasonality and a slight upward trend."
5. VLM returns anchors:
   [(0, 245.2, "rising"), (6, 312.8, "stable"), (12, 289.1, "falling"),
    (18, 265.0, "rising"), (24, 248.3, "rising"), ...]
6. Reconstruct latent trajectory via anchor-conditioned ODE solver
   extending 48 steps beyond the observation window
7. Route to forecasting tool chain: patch_embed -> self_attention ->
   linear_projection
8. Decode and invert normalization to produce 48 forecast values
9. Validate: forecast respects 24h periodicity from anchors, maintains
   upward trend bias

Output:
- forecast_values: np.array of shape (48,) with hourly predictions
- anchor_summary: "Forecast anchored by daily cycle peaks at ~06:00
  (312 kWh) and troughs at ~00:00 (245 kWh), with 0.3% daily increase"
- confidence_interval: 95% bounds derived from ODE trajectory variance
```

**Example 2: Anomaly detection in server metrics**

```
User: Detect anomalies in this CPU utilization time series from our
production server (5-minute intervals, 2 weeks of data).

Approach:
1. Load 4032-point series, normalize, note: 5-min sampling = 288/day
2. Compute priors: daily periodicity confirmed (FFT peak at 288),
   mean=42.3%, std=18.7%, ADF p=0.01 (stationary around daily cycle)
3. Plot full series with rolling mean/std bands (window=288)
4. Send to VLM: "Identify regime changes, unusual spikes, and
   structural breaks in this CPU utilization plot. Mark each with
   (time_index, value, behavior_change)."
5. VLM identifies anchors:
   [(1440, 95.2, "rising"),   -- sudden spike on day 5
    (1444, 91.8, "stable"),   -- sustained high utilization
    (1452, 38.1, "falling"),  -- return to normal
    (2880, 87.5, "rising"),   -- another spike on day 10
    (2884, 89.2, "stable"),
    (2890, 35.7, "falling")]
6. Route to anomaly detection chain: patch_embed -> multi_scale_scorer
7. Score each timestep; anchors at regime changes boost anomaly scores
8. Apply threshold (3-sigma from rolling baseline) to produce binary mask

Output:
- anomaly_mask: boolean array, True at indices [1438-1454, 2878-2892]
- anomaly_scores: continuous scores per timestep
- explanation: "Two anomalous episodes detected: Day 5 (indices 1438-1454,
  peak CPU 95.2%) and Day 10 (indices 2878-2892, peak 89.2%). Both show
  sharp rise -> sustained plateau -> sharp recovery pattern consistent
  with batch job or DDoS events."
```

**Example 3: Classifying ECG signal patterns**

```
User: Classify this ECG segment as normal or one of 4 arrhythmia types.

Approach:
1. Load single-channel ECG segment (500 samples at 125Hz = 4 seconds)
2. Compute priors: dominant frequency ~1.2Hz (72 BPM), QRS complex
   width estimate, R-R interval variability
3. Plot ECG with amplitude grid, mark detected R-peaks
4. Send to VLM: "Analyze this ECG waveform. Identify P-waves, QRS
   complexes, T-waves, and any morphological anomalies. Report anchor
   points at each significant waveform feature."
5. VLM extracts anchors at R-peaks, P-wave onsets, ST-segment levels,
   noting "ST elevation at index 180" and "widened QRS at index 220"
6. Route to classification chain: patch_embed -> temporal_conv_block ->
   global_avg_pool -> 5-class_classifier
7. Anchors indicating ST elevation bias classification toward
   myocardial infarction pattern
8. Output class probabilities with anchor-based justification

Output:
- predicted_class: "ST-elevation MI" (confidence: 0.87)
- anchor_evidence: "ST segment elevated 2.1mm above baseline at
  indices 178-195; QRS duration 142ms (normal <120ms)"
- class_probabilities: {normal: 0.04, afib: 0.03, stemi: 0.87,
  svt: 0.02, vt: 0.04}
```

## Best Practices

- **Do:** Always compute statistical priors before generating plots. The priors condition both the plot rendering (what to annotate) and the VLM prompt (what to look for). Without priors, anchor extraction is unfocused and noisy.
- **Do:** Use structured VLM prompts that specify the exact output format for anchors -- `(time_index, value, behavior)` tuples. Unstructured responses require fragile parsing and produce inconsistent results.
- **Do:** Gate agent contributions by confidence. If the VLM returns low-confidence anchors (hedging language, wide value ranges), downweight their influence on latent reconstruction rather than trusting them equally.
- **Do:** Validate outputs against anchor-implied constraints before returning results. A forecast that contradicts every extracted anchor likely has an error in the reconstruction pipeline.
- **Avoid:** Skipping the visual reasoning step and feeding raw numbers directly to the latent ODE. The anchors provide critical structural regularization -- without them, the ODE may produce smooth but structurally wrong trajectories.
- **Avoid:** Extracting too many anchors (>20 for a typical series). Over-anchoring over-constrains the latent trajectory, reducing the model's ability to capture dynamics between anchor points. Target 5-15 anchors for most series lengths.
- **Avoid:** Using the same plot style for all tasks. Forecasting plots should emphasize the prediction horizon boundary; anomaly plots should show statistical bands; classification plots should highlight morphological features.

## Error Handling

| Problem | Cause | Solution |
|---------|-------|----------|
| VLM returns no anchors | Plot too cluttered or prompt too vague | Simplify plot (subsample if >2000 points), add statistical annotations, make prompt more specific about what features to identify |
| VLM anchor values don't match actual data | Scale mismatch between plot axes and VLM's value estimates | Include explicit axis labels with numeric ticks in the plot; add ground-truth value ranges to the prompt context |
| Latent ODE diverges | Anchor constraints are contradictory or too aggressive | Reduce anchor weight (lambda parameter), remove outlier anchors that conflict with statistical priors, increase ODE solver tolerance |
| Router selects wrong tool chain | Task type ambiguous or priors misleading | Explicitly specify task type rather than relying on inference; verify priors are computed on clean (non-corrupted) data |
| Output violates physical constraints | Model predictions outside valid range (e.g., negative energy consumption) | Apply domain-specific clipping as post-processing; add physical constraints to the verification step |
| Shared memory corruption across iterations | High-confidence noise from one agent overwriting valid state | Lower the gating threshold; add memory decay (exponential moving average) so stale entries lose influence |

## Limitations

- **VLM dependency.** The quality of temporal anchors depends entirely on the VLM's ability to interpret time series plots. VLMs can misread dense, overlapping multi-variate plots or miss subtle patterns that are statistically significant but visually imperceptible.
- **Not real-time.** The plot rendering -> VLM inference -> latent ODE pipeline introduces latency incompatible with sub-second streaming requirements. Use direct numeric methods for real-time applications.
- **Univariate bias.** Visual reasoning works best on single-channel or low-dimensional series that produce readable plots. For high-dimensional multivariate series (50+ channels), the visual step either requires channel selection heuristics or becomes a bottleneck.
- **Anchor subjectivity.** Different VLM calls on the same plot may return different anchors. For production systems, consider ensembling multiple VLM passes or using deterministic feature extraction as a fallback.
- **Training still required.** While the VLM prompting is training-free, the patch embeddings, communication modules, ODE dynamics network, and routing parameters require optimization on task-specific data. This is not a zero-shot solution for novel domains without fine-tuning.

## Reference

Ruan, W. & Liang, Y. (2026). *Visual Reasoning over Time Series via Multi-Agent System.* arXiv:2602.03026v1. Key sections: Section 3 (Analyzer-Reasoner-Executor architecture), Section 3.2 (anchor extraction and latent ODE reconstruction), Section 3.3 (gated communication and routing), Table 1-4 (benchmark results across forecasting, classification, anomaly detection, imputation).