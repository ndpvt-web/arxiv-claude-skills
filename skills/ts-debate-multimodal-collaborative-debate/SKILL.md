---
name: "ts-debate-multimodal-collaborative-debate"
description: "Zero-shot time series reasoning via modality-specialized multi-agent debate. Assigns dedicated text, visual, and numerical analyst agents to reason over temporal data, coordinated by structured debate and reviewer verification. Use when: 'analyze this time series data', 'classify this sensor signal', 'forecast this trend from context and charts', 'detect anomalies in this dataset', 'answer questions about this temporal data', 'reason over these multiple modalities together'."
---

# TS-Debate: Multimodal Collaborative Debate for Zero-Shot Time Series Reasoning

This skill enables Claude to reason over time series data by orchestrating a structured multi-agent debate across three modalities: textual context, visual patterns, and numerical signals. Instead of collapsing all information into a single prompt (which causes modality interference and numeric hallucinations), the TS-Debate protocol isolates each modality into a dedicated analyst role, runs a collaborative debate with explicit verification, and synthesizes a calibrated final answer. This approach achieves strong zero-shot performance on classification, regression, forecasting, anomaly detection, and temporal QA tasks without any fine-tuning.

## When to Use

- When the user provides time series data (CSV, DataFrame, or numeric arrays) and asks to classify, forecast, or detect anomalies
- When the user supplies both a chart/plot and raw numbers and wants reasoning that accounts for both
- When a question requires combining domain context (e.g., "this is ECG data" or "these are stock prices") with quantitative analysis
- When the user asks to answer questions about temporal patterns, trends, seasonality, or correlations
- When the user wants to avoid numeric hallucinations in LLM-based time series analysis
- When multiple signals or modalities (text descriptions, images, tables) describe the same temporal phenomenon and need cross-validated reasoning

## Key Technique

**Modality isolation prevents interference.** Standard approaches either serialize all modalities into one long text prompt (causing token overflow and numeric drift) or fuse them implicitly in a single model pass (enabling undetected cross-modal contamination). TS-Debate instead assigns three specialist agents -- a Text Analyst, a Visual Analyst, and a Numerical Analyst -- each receiving only their modality-specific representation. Disagreements between agents therefore reflect genuine evidence differences rather than prompt artifacts.

**Domain knowledge elicitation creates a shared contract.** Before any data analysis, the system elicits task-specific priors: what domain is this (finance, health, energy)? What constraints apply? What patterns matter? What failure modes are common? Which modalities are most informative? This "shared analysis contract" anchors all subsequent reasoning to domain-appropriate methodology, preventing analysts from drifting into generic pattern-matching.

**Verification-Conflict-Calibration (VCC) replaces voting.** Reviewer agents evaluate each analyst's claims by assigning verification status (VERIFIED / UNVERIFIED / CONTRADICTED), checking domain consistency, and scoring inference quality, observation specificity, and intellectual honesty on a 0-100 scale. Conflicts are explicitly modeled as NO_CONFLICT, DETECTED, or RESOLVED. A final synthesizer prioritizes answers backed by verified, domain-consistent evidence rather than counting votes -- so a single well-verified numerical claim can outweigh two vague visual impressions.

## Step-by-Step Workflow

1. **Elicit domain knowledge.** Given the user's query and any metadata, generate a domain knowledge brief: characterize the task domain, list expected temporal patterns (seasonality, trends, regime changes), identify which modalities are most informative, note common failure modes (e.g., confusing noise for signal in ECG, overfitting to recent trend in finance), and specify recommended analysis procedures.

2. **Separate input modalities.** Partition the available data into three channels:
   - **Text**: domain descriptions, metadata, column headers, units, any narrative context the user provides.
   - **Visual**: time-domain plots, frequency spectra, heatmaps, or any chart. If none exist, generate a matplotlib plot from the raw data.
   - **Numerical**: the raw numeric values, summary statistics (mean, std, min, max, percentiles), and derived features (autocorrelation, first differences, rolling averages).

3. **Run Text Analyst (Round 1).** Analyze only the textual context and domain knowledge. Produce structured evidence: (a) Understanding -- restate the task, (b) Observations -- specific checkable claims grounded in text, (c) Inferences -- conclusions drawn from observations, (d) Limits -- what cannot be determined from text alone.

4. **Run Visual Analyst (Round 1).** Analyze only the chart(s) and domain knowledge. Identify global structure: overall trend direction, visible seasonality/periodicity, anomalous regions, regime changes, distribution shape. Produce the same structured evidence format. Do not reference specific numeric values -- describe patterns qualitatively.

5. **Run Numerical Analyst (Round 1).** Analyze only the raw numbers and domain knowledge. Compute precise statistics, apply programmatic checks (write and execute short Python snippets for aggregations, correlations, statistical tests). Produce checkable numeric claims with exact values. Flag any values that seem inconsistent or suspicious.

6. **Cross-modal debate (Round 2).** Each analyst receives the other two analysts' Round 1 evidence. Each analyst refines their position: confirm, strengthen, or revise claims in light of cross-modal evidence. Analysts retain principled disagreements when their modality-specific evidence supports it -- do not force consensus.

7. **Reviewer verification (VCC).** For each analyst's final claims, assign verification status:
   - **VERIFIED**: claim is directly supported by at least one other modality or programmatic check.
   - **UNVERIFIED**: insufficient cross-modal evidence to confirm or deny.
   - **CONTRADICTED**: claim is directly refuted by another modality's verified evidence.
   Score each analyst on inference quality, observation specificity, and acknowledged limitations (0-100 each). Classify conflicts as NO_CONFLICT, DETECTED, or RESOLVED.

8. **Programmatic spot-checks.** For any numeric claim marked UNVERIFIED or CONTRADICTED, write and execute a short Python snippet to verify or refute it directly. Update verification status based on execution results.

9. **Synthesize final answer.** Prioritize conclusions backed by VERIFIED, domain-consistent evidence. Weight numerical precision over visual impression when they conflict on quantities. Weight visual pattern over numerical noise when assessing global structure. Produce a calibrated answer that states confidence level and flags any unresolved conflicts.

10. **Present result with audit trail.** Return the final answer to the user along with a concise summary of which modalities agreed, where conflicts arose, and how they were resolved.

## Concrete Examples

**Example 1: Stock Price Trend Classification**

```
User: Here's 30 days of AAPL closing prices [provides CSV]. Is the overall
trend bullish, bearish, or sideways? I also have this candlestick chart
[attaches image].

Approach:
1. Domain knowledge: Finance/equities. Key patterns: support/resistance,
   moving average crossovers, volume confirmation. Failure mode: recency
   bias (weighting last few days over the full 30-day window).

2. Text Analyst: "The user asks about 30-day trend for AAPL. No earnings
   or macro events mentioned. The time horizon suggests a swing-trading
   perspective."

3. Visual Analyst: "The candlestick chart shows a series of higher lows
   from day 1-20, followed by a consolidation with smaller bodies from
   day 20-30. No clear breakdown visible. Overall structure: bullish
   with recent deceleration."

4. Numerical Analyst (executes Python):
   - 30-day return: +4.2%
   - Linear regression slope: +0.31/day (p < 0.01)
   - 10-day MA crossed above 20-day MA on day 12, still above
   - Last 10 days: range-bound, daily returns std = 0.4%
   Conclusion: "Statistically bullish trend with reduced momentum in
   the final third."

5. Round 2: Visual and Numerical analysts agree on bullish-with-deceleration.
   Text analyst notes no contradicting context.

6. VCC: All three claims VERIFIED, NO_CONFLICT.

Output:
The 30-day trend is **bullish with recent deceleration**.
- Supported by: +4.2% total return, positive regression slope (p<0.01),
  bullish MA crossover on day 12 (Numerical, VERIFIED)
- Confirmed by: higher-low pattern with consolidation in final 10 days
  (Visual, VERIFIED)
- No contradicting textual context
- Confidence: High. Unresolved: whether consolidation is a pause or
  reversal -- monitor for breakdown below 20-day MA.
```

**Example 2: ECG Anomaly Detection**

```
User: I have a 10-second ECG recording at 250Hz [provides array of 2500
values]. Are there any anomalies? The patient is a 55-year-old male with
no known cardiac history.

Approach:
1. Domain knowledge: Cardiology/ECG. Normal sinus rhythm has regular R-R
   intervals, consistent P-QRS-T morphology. Anomalies: PVCs (premature
   ventricular contractions), ST elevation/depression, atrial fibrillation
   (irregular R-R). Failure mode: confusing baseline wander artifacts
   with true ST changes.

2. Text Analyst: "55-year-old male, no known history. Age is a risk
   factor for cardiac events but no prior diagnosis. No medications
   mentioned that could alter ECG morphology."

3. Visual Analyst (from generated plot): "Regular rhythm visible for most
   of the trace. One complex near t=6.2s appears wider and morphologically
   different from surrounding beats. No obvious ST segment deviation.
   Baseline is stable."

4. Numerical Analyst (executes Python):
   - Detected 14 R-peaks via scipy.signal.find_peaks
   - R-R intervals: [0.72, 0.71, 0.73, 0.72, 0.71, 0.58, 0.85, 0.72,
     0.71, 0.73, 0.72, 0.71, 0.72] seconds
   - Beat 6 (at ~6.1s): R-R shortened to 0.58s, followed by compensatory
     pause of 0.85s
   - QRS width of beat 6: ~140ms vs normal ~90ms for other beats
   Conclusion: "Beat 6 is a premature ventricular contraction (PVC).
   Single isolated PVC in a 10s strip."

5. Round 2: Visual analyst confirms the wider complex at t=6.2s matches
   the numerically identified PVC. Text analyst notes PVCs can be benign
   but age warrants follow-up.

6. VCC: PVC identification VERIFIED by both visual and numerical.
   Compensatory pause VERIFIED by R-R interval calculation.

Output:
**One anomaly detected: isolated premature ventricular contraction (PVC)
at approximately t=6.1s.**
- Evidence: shortened R-R interval (0.58s vs 0.72s baseline), widened
  QRS (~140ms vs ~90ms), followed by compensatory pause (0.85s)
  (Numerical, VERIFIED; Visual, VERIFIED)
- Clinical note: isolated PVCs can be benign, but given patient age (55),
  clinical correlation is recommended
- Confidence: High for PVC identification. No other anomalies detected.
```

**Example 3: Energy Consumption Forecasting QA**

```
User: Given this building's hourly energy consumption for the past week
[provides 168 values], what do you expect consumption to be during
business hours (9am-5pm) next Monday? The building is an office in
a temperate climate zone.

Approach:
1. Domain knowledge: Building energy/HVAC. Strong daily periodicity
   (high during business hours, low overnight). Weekly cycle (weekday
   vs weekend). Temperature sensitivity exists but climate is temperate.
   Failure mode: not accounting for Monday potentially differing from
   mid-week patterns.

2. Text Analyst: "Office building, temperate climate. Business hours
   9am-5pm. Expect HVAC ramp-up around 7-8am, peak mid-afternoon,
   decline after 5pm. Monday may show a 'cold start' effect -- slightly
   higher than mid-week as HVAC compensates for weekend setback."

3. Visual Analyst (from plot): "Clear daily sawtooth pattern on weekdays
   with peaks around 2-3pm. Weekend consumption is ~40% of weekday peak.
   All five weekdays appear similar in shape and magnitude."

4. Numerical Analyst (executes Python):
   - Weekday 9am-5pm averages: Mon=245, Tue=238, Wed=241, Thu=239, Fri=236 kWh/hr
   - Weekend 9am-5pm average: 98 kWh/hr
   - Monday average is 2.6% higher than other weekdays (cold start effect)
   - Hour-by-hour Monday profile: ramps from 180 (9am) to 290 (2pm),
     declines to 220 (5pm)
   Forecast: "Next Monday 9am-5pm total: ~1,960 kWh (245 kWh/hr x 8 hrs)"

5. Round 2: Visual confirms weekday consistency. Text analyst's cold
   start hypothesis is VERIFIED by Monday's slightly elevated average.

6. VCC: Forecast VERIFIED. Monday cold-start VERIFIED numerically.

Output:
**Expected next Monday 9am-5pm: ~1,960 kWh total (~245 kWh/hr average)**
- Based on this week's Monday profile: ramp from ~180 kWh/hr (9am) to
  ~290 kWh/hr peak (2pm), declining to ~220 kWh/hr (5pm)
- Monday runs ~2.6% above Tue-Fri average (cold-start effect confirmed
  in historical data)
- Confidence: Moderate-High. Assumes no significant weather change or
  occupancy deviation. Temperature sensitivity not assessable from one
  week of data.
```

## Best Practices

- **Do:** Keep modalities strictly separated in Round 1. The whole point is that each analyst reasons from their own evidence first. Premature mixing defeats the purpose.
- **Do:** Write and execute actual Python code for numeric verification. A 3-line pandas snippet catches hallucinations that verbal reasoning misses.
- **Do:** Preserve principled disagreements. If the visual pattern says "uptrend" but the numbers say "statistically flat," report both with verification status rather than forcing agreement.
- **Do:** Tailor domain knowledge elicitation to the specific domain. ECG analysis needs different priors than stock price analysis or weather forecasting.
- **Avoid:** Letting the numerical analyst make qualitative claims ("looks seasonal") without quantitative backing (autocorrelation at lag 24 = 0.87).
- **Avoid:** Treating the final synthesis as a majority vote. One VERIFIED numerical claim outweighs two UNVERIFIED visual impressions.
- **Avoid:** Skipping the domain knowledge step. Without it, analysts default to generic pattern matching and miss domain-specific failure modes (e.g., baseline wander in ECG, stock splits in finance).

## Error Handling

- **Missing modality**: If no chart/image is available, generate one from raw data using matplotlib. If no raw numbers are available (only a chart), the Numerical Analyst should note this limitation and the Visual Analyst takes priority.
- **Conflicting verified claims**: When two modalities produce VERIFIED but contradictory conclusions, escalate: run additional programmatic checks, examine the data at finer granularity, and report the conflict explicitly with both pieces of evidence.
- **Code execution failure**: If a Python verification snippet fails (import error, data format issue), mark the claim UNVERIFIED rather than CONTRADICTED. Attempt a simpler computation as fallback.
- **Ambiguous domain**: If the domain isn't clear from context, ask the user. Domain knowledge elicitation is foundational -- guessing the wrong domain (e.g., treating seismic data as financial data) corrupts the entire analysis.
- **Insufficient data**: For very short time series (<10 points), note that statistical tests and frequency analysis are unreliable. Lean on domain knowledge and qualitative visual assessment, flagging low confidence.

## Limitations

- **Not a replacement for trained models.** On large-scale forecasting tasks where specialized models (N-BEATS, PatchTST, TimesFM) are available and trained, TS-Debate will underperform. Its strength is zero-shot reasoning when no trained model exists for the specific task.
- **Cost scales with debate rounds.** The full protocol (3 analysts x 2 rounds + 3 reviewers + synthesizer) requires multiple LLM calls. For batch processing thousands of time series, this is expensive (~$0.03/sample with gpt-4.1-mini).
- **Image understanding limits.** Visual analysis depends on chart quality. Cluttered plots, unlabeled axes, or very dense time series produce weaker visual evidence.
- **Numerical precision ceiling.** LLM-based numerical analysis, even with code execution, is bounded by the precision of the data provided. Floating-point artifacts in user data propagate through analysis.
- **Single-series focus.** The protocol handles one time series (or a small set of related series) per invocation. Large multivariate panel analysis with dozens of correlated series requires adaptation.

## Reference

- **Paper**: [TS-Debate: Multimodal Collaborative Debate for Zero-Shot Time Series Reasoning](https://arxiv.org/abs/2601.19151v1) (Trirat et al., 2026)
- **Key insight**: Modality-specialized agents that debate and verify claims through structured VCC protocol outperform standard multimodal fusion by +25.65% on classification and 36.92% lower MAE on regression tasks across 20 time series benchmarks.
- **Code**: https://github.com/DeepAuto-AI/TS-Debate