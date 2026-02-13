---
name: "tsaqa-time-series-analysis"
description: "Structured time series question answering using the TSAQA six-task framework: anomaly detection, classification, characterization, comparison, data transformation, and temporal relationship analysis. Triggers: 'analyze this time series', 'detect anomalies in sensor data', 'compare these two time series', 'classify this signal', 'what trend does this data show', 'reorder these time segments'"
---

# TSAQA: Structured Time Series Analysis via Question Answering

This skill enables Claude to perform rigorous time series analysis by decomposing requests into six well-defined analytical tasks drawn from the TSAQA benchmark framework. Rather than treating time series analysis as a monolithic problem, TSAQA structures it into anomaly detection, classification, characterization (trend/seasonality/dispersion), comparison, data transformation understanding (Fourier, wavelet, differencing), and temporal relationship analysis. This structured decomposition, combined with z-score normalization and patch-based reasoning, produces more reliable answers than ad-hoc analysis -- particularly for the characterization, comparison, and ordering tasks where LLMs otherwise show smoothness bias and poor frequency-domain reasoning.

## When to Use

- When the user provides numerical time series data and asks questions about patterns, anomalies, or behavior
- When comparing two or more time series for similarity, correlation, or divergence
- When the user asks to detect anomalies or outliers in sensor, financial, or operational data
- When classifying a time series signal into a known category (e.g., ECG arrhythmia type, activity recognition)
- When the user needs to understand the effect of transformations like FFT, wavelet decomposition, or differencing on their data
- When reordering shuffled time segments into correct chronological sequence
- When the user asks about trend direction, seasonality period, or volatility characteristics of a series
- When building or evaluating time series QA datasets or prompts for LLM-based analysis

## Key Technique

**Six-Task Decomposition.** TSAQA's core insight is that time series "understanding" is not one capability but six distinct ones. Conventional tasks -- anomaly detection (is this point abnormal?) and classification (what type of signal is this?) -- are well-studied. The benchmark adds four advanced tasks: *characterization* decomposes a series into trend, seasonality, and dispersion properties; *comparison* evaluates relative similarity between two series using metrics like DTW and Pearson correlation; *data transformation* tests whether a model understands what Fourier transforms, wavelet decompositions, or differencing operations do to a signal; and *temporal relationship* requires reconstructing the correct chronological order of shuffled subsequences (patches).

**Normalization and Patching.** All time series are z-score normalized before analysis (subtract mean, divide by standard deviation) to eliminate scale bias across domains. For temporal relationship tasks, series are split into non-overlapping patches of equal length, shuffled, and the model must reconstruct the original ordering. This patches approach directly tests whether the model can reason about local continuity and transition dynamics rather than just global statistics.

**Question Format Design.** Three formats probe different reasoning depths: True/False (TF) for binary claims, Multiple Choice (MC) for discrimination among plausible alternatives, and Puzzling (PZ) for open-ended temporal reconstruction. The PZ format -- where the model receives a first patch and must order the remaining shuffled patches -- is the most challenging, with even instruction-tuned models reaching only ~68% accuracy. This reveals a critical weakness: LLMs exhibit *smoothness bias*, preferring transitions that are artificially smooth rather than matching actual discontinuities in the data.

## Step-by-Step Workflow

1. **Normalize the input.** Z-score the raw time series: subtract the mean and divide by the standard deviation. If multiple series are involved, normalize each independently. This removes domain-specific scale effects and makes cross-domain reasoning consistent.

2. **Identify the task type.** Map the user's question to one of the six TSAQA tasks:
   - "Is this point/segment unusual?" → Anomaly Detection
   - "What type of signal is this?" → Classification
   - "What are the trend/seasonality/volatility properties?" → Characterization
   - "How similar are series A and B?" → Comparison
   - "What happens after applying FFT/wavelet/differencing?" → Data Transformation
   - "What is the correct order of these segments?" → Temporal Relationship

3. **Choose the question format.** Use TF for quick validation of specific claims, MC when presenting differential diagnoses or competing hypotheses, and PZ (puzzling/ordering) only for temporal reconstruction tasks.

4. **Serialize the time series as text.** Convert numerical values to a comma-separated string with consistent decimal precision (2-4 places). For series longer than 256 points, subsample or segment into patches of 32-256 values. Prefix with contextual metadata: domain, sampling rate, units if known.

5. **Construct the structured prompt.** Combine contextual information C, the serialized time series T, and the question Q in this order:
   ```
   Context: [domain, sampling info, units]
   Time Series: [comma-separated normalized values]
   Question: [specific analytical question]
   Answer format: [TF | MC with options | PZ ordering]
   ```

6. **Apply task-specific analysis.** For each task type, use the appropriate analytical lens:
   - *Anomaly Detection*: Check values beyond 3x IQR or sudden distribution shifts
   - *Classification*: Extract discriminative features (shape, frequency content, amplitude patterns)
   - *Characterization*: Decompose into trend (direction, slope), seasonality (period, amplitude), dispersion (variance, range)
   - *Comparison*: Compute similarity via correlation, DTW distance, or shape alignment
   - *Data Transformation*: Reason about frequency content (FFT), multi-scale features (wavelet), or stationarity (differencing)
   - *Temporal Relationship*: Check patch boundary continuity -- values at the end of one patch should be consistent with the start of the next

7. **Guard against smoothness bias.** When ordering patches or detecting anomalies, do not assume transitions must be smooth. Real time series contain legitimate discontinuities, regime changes, and abrupt shifts. Evaluate boundary compatibility based on the series' observed local dynamics, not idealized smoothness.

8. **Validate with cross-checks.** For characterization and comparison tasks, generate the answer, then verify by asking the inverse question. If you claim "upward trend," confirm that a downward-trend claim would be false. For transformation tasks, verify that stated properties (e.g., "dominant frequency at 0.1 Hz") are consistent with the raw data's periodicity.

9. **Present results with confidence and evidence.** State the answer, the reasoning chain, and the specific data points or segments that support it. Flag uncertainty explicitly -- especially for domains like web traffic and sales where volatility makes analysis inherently harder.

10. **Generate code when appropriate.** If the user needs reproducible analysis, produce Python code using numpy/scipy/pandas for the computation, with the TSAQA task structure as the organizing principle.

## Concrete Examples

**Example 1: Anomaly Detection in Sensor Data**

User: "Here's 100 temperature readings from a server room sensor. Are there any anomalies? Values: 22.1, 22.3, 22.0, 22.4, 22.2, ... 38.7, 22.1, 22.3 ..."

Approach:
1. Z-score normalize the series (mean ~22.2, std ~0.3 excluding the spike)
2. Identify task: Anomaly Detection (TF format -- is there an anomaly?)
3. Compute IQR-based thresholds: Q1=22.0, Q3=22.4, IQR=0.4, upper fence=22.4 + 3*0.4 = 23.6
4. Flag value 38.7 at position 67 as anomalous (16.5 standard deviations above mean)
5. Check for contextual anomalies: verify surrounding values return to baseline

Output:
```
Anomaly detected at index 67: value 38.7 (z-score: +54.7).

This is a point anomaly -- the series returns to baseline immediately
(22.1 at index 68). The remaining 99 values fall within [21.8, 22.6],
consistent with stable server room temperature.

Likely cause: sensor glitch or momentary thermal event.

import numpy as np
data = np.array([...])  # your 100 values
z_scores = (data - data.mean()) / data.std()
anomalies = np.where(np.abs(z_scores) > 3)[0]
# anomalies: [67]
```

**Example 2: Comparing Two Financial Time Series**

User: "Compare these two stock return series and tell me if they move together. Series A: [0.02, -0.01, 0.03, -0.02, 0.01, ...] Series B: [0.01, -0.03, 0.02, -0.01, 0.02, ...]"

Approach:
1. Z-score normalize each series independently
2. Identify task: Comparison
3. Compute Pearson correlation between normalized series
4. Compute DTW distance for shape-based similarity
5. Assess lead-lag relationship by cross-correlation
6. Frame answer as MC: "highly correlated / moderately correlated / uncorrelated / inversely correlated"

Output:
```
These series are moderately positively correlated (Pearson r = 0.61, p < 0.01).

Characterization:
- Both show similar directional movement in 73% of periods
- Series B lags Series A by approximately 1 period (cross-correlation peak at lag=1)
- DTW distance: 0.34 (moderate shape similarity)

They move together directionally but with different magnitudes and a
slight lag, consistent with a lead-follow relationship.

import numpy as np
from scipy import stats
from dtaidistance import dtw
a, b = np.array([...]), np.array([...])
a_z = (a - a.mean()) / a.std()
b_z = (b - b.mean()) / b.std()
r, p = stats.pearsonr(a_z, b_z)
dtw_dist = dtw.distance(a_z, b_z)
```

**Example 3: Temporal Relationship -- Reordering Shuffled Segments**

User: "I have a time series split into 4 patches that got shuffled. Patch 1 (anchor): [2.1, 2.3, 2.5, 2.7]. Shuffled patches -- A: [4.1, 4.3, 4.5, 4.7], B: [3.1, 3.3, 3.5, 3.7], C: [2.9, 3.0, 2.8, 3.1]. What's the correct order?"

Approach:
1. Identify task: Temporal Relationship (PZ format)
2. The anchor patch ends at 2.7 with a clear upward linear trend
3. Check boundary continuity: which patch starts closest to where patch 1 ends (2.7)?
4. Patch C starts at 2.9 -- smooth continuation from 2.7
5. Patch C ends around 3.1; Patch B starts at 3.1 -- perfect continuation
6. Patch B ends at 3.7; Patch A starts at 4.1 -- slight jump but closest remaining option
7. Verify: the full reconstructed series [2.1...2.7, 2.9...3.1, 3.1...3.7, 4.1...4.7] shows a consistent upward trend

Output:
```
Correct chronological order: Patch 1 → C → B → A

Reasoning (boundary continuity analysis):
- Patch 1 ends at 2.7 → Patch C starts at 2.9 (gap: 0.2, consistent with trend slope)
- Patch C ends at ~3.1 → Patch B starts at 3.1 (gap: 0.0, seamless)
- Patch B ends at 3.7 → Patch A starts at 4.1 (gap: 0.4, largest but only remaining option)

The reconstructed series shows a monotonically increasing trend from
2.1 to 4.7 with minor local variation in Patch C.
```

## Best Practices

- **Do:** Always z-score normalize before analysis. Domain-specific scales (e.g., temperature in Celsius vs. stock prices in dollars) introduce spurious difficulty. Normalization makes the analysis focus on shape and pattern, not magnitude.

- **Do:** Explicitly state which of the six task types you are performing. This focuses the analysis and prevents conflating different analytical questions (e.g., mistaking a classification question for an anomaly detection one).

- **Do:** For comparison tasks, use multiple similarity metrics (Pearson, DTW, cross-correlation) rather than relying on a single measure. Different metrics capture different aspects of similarity.

- **Do:** When handling series longer than 256 values, segment into patches and analyze both local (per-patch) and global (cross-patch) properties. Report which patches drive the overall conclusion.

- **Avoid:** Assuming smooth transitions between segments. Real time series contain regime changes, level shifts, and structural breaks. The TSAQA benchmark shows LLMs consistently over-smooth, predicting artificially gradual transitions. Trust the data's actual dynamics.

- **Avoid:** Attempting Fourier or wavelet transform reasoning from raw text values alone. LLMs perform near-random on frequency-domain questions in zero-shot settings (except for first-order differencing). When transformation analysis is needed, generate executable code to compute the actual transform rather than reasoning about it abstractly.

## Error Handling

- **Too few data points (<32):** Warn that statistical characterization (trend, seasonality) is unreliable. Require at least 2 full periods for seasonality detection.
- **Missing values >1%:** Flag data quality concern. Impute only with explicit user consent, using linear interpolation for small gaps or domain-appropriate methods for larger ones.
- **Extreme outliers >5% of series:** The series may have systematic issues rather than point anomalies. Suggest the user verify data collection before anomaly analysis.
- **Ambiguous task mapping:** If the user's question spans multiple task types (e.g., "compare these series and detect anomalies in each"), decompose into separate sub-tasks and address each with the appropriate framework.
- **Transformation questions without code execution:** If asked about Fourier or wavelet properties and you cannot run code, state explicitly that text-based reasoning about frequency-domain properties is unreliable and provide executable code the user can run.

## Limitations

- **Frequency-domain reasoning is weak.** Both commercial and open-source LLMs perform poorly on Fourier and wavelet transformation questions when reasoning from text alone. Always generate code for these tasks rather than attempting purely verbal analysis.
- **Temporal ordering degrades with more patches.** Accuracy drops significantly as the number of shuffled patches increases. For sequences with more than 6-8 patches, computational approaches (e.g., greedy boundary matching) are more reliable than pure reasoning.
- **Domain-specific volatility.** Web traffic and sales data domains show the lowest analysis accuracy due to high stochasticity and weak temporal structure. Flag increased uncertainty for these domains.
- **Long series challenge.** Serializing more than ~256 numerical values as text risks both context window pressure and degraded attention to individual values. Segment long series into patches.
- **No causal inference.** TSAQA tasks assess correlation, pattern, and ordering -- not causation. Do not make causal claims based on time series co-movement alone.

## Reference

**Paper:** Jing, B., Chen, S., Zheng, L., Liu, B., & Li, Z. (2026). *TSAQA: Time Series Analysis Question And Answering Benchmark.* arXiv:2601.23204v1. https://arxiv.org/abs/2601.23204v1

**Key takeaway:** The six-task decomposition framework (anomaly detection, classification, characterization, comparison, data transformation, temporal relationship) combined with z-score normalization and patch-based reasoning provides a structured approach to time series QA that outperforms ad-hoc analysis. Pay particular attention to the smoothness bias finding and the near-random performance on frequency-domain tasks -- these directly inform when to trust reasoning vs. when to generate code.

**Dataset:** https://huggingface.co/datasets/TSAQA/TSAQA-Benchmark (210k samples across 13 domains)