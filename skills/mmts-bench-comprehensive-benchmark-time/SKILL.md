---
name: "mmts-bench-comprehensive-benchmark-time"
description: "Evaluate and improve LLM performance on time series question-answering using the MMTS-BENCH hierarchical taxonomy. Covers structural awareness, feature analysis, temporal reasoning, sequence matching, and cross-modal alignment. Use when: 'evaluate time series reasoning', 'benchmark LLM on temporal data', 'build time series QA pipeline', 'assess model on trend/seasonality/anomaly tasks', 'create time series understanding tests', 'improve LLM time series accuracy'."
---

# MMTS-BENCH: Systematic Time Series Understanding and Reasoning for LLMs

This skill enables Claude to build, evaluate, and improve time series question-answering (TSQA) systems using the hierarchical taxonomy and methodology from MMTS-BENCH. It provides a structured framework for decomposing time series understanding into five testable dimensions — structural awareness, feature analysis, temporal reasoning, sequence matching, and cross-modal alignment — and applying evidence-backed strategies (statistical prefixing, chain-of-thought prompting, multimodal input) to maximize LLM accuracy on temporal data tasks.

## When to Use

- When a user wants to evaluate how well an LLM understands time series data (trend detection, seasonality, anomalies, stationarity)
- When building a TSQA benchmark or test suite for temporal reasoning capabilities
- When designing prompts that feed time series data to LLMs and need to maximize accuracy
- When the user asks to create a pipeline that generates questions about time series datasets
- When comparing LLM approaches to time series analysis (text-only vs. multimodal, with/without CoT)
- When building a time series encoder + LLM alignment system and need to evaluate it rigorously
- When diagnosing why an LLM fails on specific temporal tasks (local vs. global, causal vs. deductive reasoning)

## Key Technique

MMTS-BENCH organizes time series understanding into a **hierarchical taxonomy** with five dimensions, each targeting a distinct reasoning capability. **Structural awareness** tests whether a model can identify stationarity, regime shifts, and local vs. global patterns. **Feature analysis** covers trend, seasonality, noise, volatility, and basic statistics. **Temporal reasoning** spans deductive, inductive, causal, analogical, and counterfactual reasoning over temporal data. **Sequence matching** evaluates morphological correspondence via isomorphic, robust (post-smoothing), localization, and reverse matching. **Cross-modal alignment** tests bidirectional conversion between time series and natural language descriptions. Cross-combining feature analysis (7 types) with temporal reasoning (5 types) yields 35 composite subtasks, plus structural, matching, and alignment tasks — totaling 286 fine-grained evaluation categories.

The benchmark's construction methodology is equally important. Synthetic data (Base subset) uses STL-inspired decomposition (trend + seasonality + noise) with 17 parameterized templates, where exact construction parameters are logged to enable graded-difficulty questions. Real-world data (InWild subset) is processed through a **three-stage pipeline**: (1) multimodal context integration that feeds raw sequences, visualizations, domain metadata, and pre-computed statistics to an LLM to produce global summaries and local descriptions; (2) reasoning-driven QA synthesis that generates Question-Answer-Explanation triples with explicit justification requirements; (3) multi-dimensional verification checking logical validity, mathematical correctness, and cross-modal consistency, followed by expert review. This staged approach significantly reduces spurious cues compared to single-turn question generation.

The most actionable finding: **statistical prompt prefixes** (providing offset, scale, min/max, length, boundary values alongside the series) and **chain-of-thought reasoning** each deliver larger accuracy gains than scaling model parameters or improving time series encoder architectures. The backbone LLM's general reasoning capacity dominates performance — encoder design (MLP, CNN, Transformer) produces only marginal differences. This means the highest-leverage improvements come from how you present the data and prompt the model, not from specialized architectures.

## Step-by-Step Workflow

1. **Classify the time series task** using the five-dimension taxonomy. Determine whether the task requires structural awareness (stationarity, regime detection), feature analysis (trend, seasonality, volatility), temporal reasoning (causal, counterfactual, deductive), sequence matching (similarity, localization), or cross-modal alignment (description generation/matching). Most real tasks combine feature analysis + temporal reasoning.

2. **Compute and prepend a statistical prefix** to the time series representation. Calculate and include: sequence length, min, max, mean, standard deviation, offset (first value), scale factor, boundary values (first and last N points), and autocorrelation at key lags. This single step recovers most of the accuracy lost from missing encoder information.

3. **Choose the input modality** based on available capabilities. If the target model supports vision, provide both a standardized plot (matplotlib line chart with grid, axis labels, consistent styling) AND the numerical text representation. Multimodal input yields ~8 percentage points improvement over text-only for capable models (e.g., Gemini 2.5 Pro: 68% text-only → 76% vision+text).

4. **Construct questions using the three-stage QA framework**. Stage 1: Generate a context block combining the statistical prefix, a natural-language summary of global patterns, and fine-grained local descriptions of notable segments. Stage 2: Generate Question-Answer-Explanation triples, parameterized by task type, scope (local/global), and dimensionality (uni/multivariate). Stage 3: Verify logical consistency, mathematical correctness, and cross-modal alignment of each QA pair.

5. **Apply chain-of-thought prompting** for temporal reasoning tasks. Explicitly instruct the model to decompose its reasoning: identify relevant features → state the temporal relationship → draw the conclusion. CoT provides larger gains on temporal reasoning than on feature analysis, and surpasses the benefit of scaling from 14B to 32B parameters.

6. **Distinguish local from global task scope**. Global tasks (overall trend direction, dominant periodicity) are substantially easier for LLMs than local tasks (pattern in a specific index range, local anomaly characterization). When a task requires local analysis, explicitly specify the index range and provide zoomed-in statistics for that window.

7. **For sequence matching tasks, use DTW-based candidate retrieval**. Compute Dynamic Time Warping distances to identify plausible candidates and distractors. Test four difficulty levels: isomorphic (same length), robust (smoothed variants), localization (subsequence within longer series), and reverse (temporally inverted). Localization and reverse matching are significantly harder.

8. **For cross-modal alignment, leverage anchor statistics**. Models succeed at TS↔text alignment primarily by matching basic statistical cues (maxima, minima, range, trend direction). Ensure distractors share overlapping domains but differ in these anchor statistics to create meaningful discrimination challenges.

9. **Evaluate with stratified metrics**. Use exact-match accuracy for categorical (multiple-choice, binary) questions. Use relative accuracy `max(1.0 - |predicted - actual| / |actual|, 0.0)` for numerical questions, plus Accuracy@10% (binary: within 10% relative error). Report results stratified by task dimension, scope (local/global), and question type.

10. **Diagnose failures using the taxonomy**. If accuracy is low on seasonality tasks (consistently the hardest feature-analysis category), investigate whether the model receives sufficient periodic context. If causal/counterfactual reasoning fails, add explicit CoT scaffolding. If local tasks underperform, add windowed statistics. Map each failure to the taxonomy to identify targeted fixes.

## Concrete Examples

**Example 1: Building a time series QA evaluation for a financial LLM**

User: "I want to test how well our fine-tuned LLM understands stock price time series. Can you create a benchmark?"

Approach:
1. Classify relevant tasks: trend detection, volatility assessment, regime change detection (structural awareness), causal reasoning ("What caused the drop at index 150-170?"), and seasonality detection.
2. Generate synthetic test series using STL decomposition:

```python
import numpy as np

def generate_ts(n=500, trend='linear', seasonal_period=50, noise_std=0.1, seed=42):
    np.random.seed(seed)
    t = np.arange(n)
    # Trend component
    if trend == 'linear':
        tr = 0.02 * t
    elif trend == 'quadratic':
        tr = 0.0001 * t**2
    # Seasonal component
    seasonal = 2.0 * np.sin(2 * np.pi * t / seasonal_period)
    # Noise
    noise = noise_std * np.random.randn(n)
    series = tr + seasonal + noise
    # Log parameters for graded questions
    params = {
        'trend_type': trend, 'seasonal_period': seasonal_period,
        'noise_std': noise_std, 'length': n,
        'min': float(series.min()), 'max': float(series.max()),
        'mean': float(series.mean()), 'std': float(series.std())
    }
    return series, params
```

3. Generate QA triples per the three-stage framework:

```
Statistical prefix: length=500, min=-2.31, max=12.05, mean=4.87, std=3.21,
first_value=0.12, last_value=9.83, autocorrelation_lag1=0.97

Q: What is the dominant seasonal period of this time series?
A: 50
Explanation: The series exhibits regular oscillations with peaks approximately
every 50 time steps, consistent with a sinusoidal seasonal component of period 50.
```

4. Include local-scope variants: "What is the trend direction between indices 200-250?"
5. Evaluate with relative accuracy for numerical answers, exact match for categorical.

**Example 2: Optimizing LLM prompts for time series anomaly detection**

User: "Our model gets 45% accuracy on time series anomaly questions. How do I improve it?"

Approach:
1. Diagnose using the taxonomy — anomaly detection spans structural awareness (local) + feature analysis (noise/volatility) + temporal reasoning (causal). Local tasks are the hardest category.
2. Add a statistical prefix to every prompt:

```
Before (45% accuracy):
"Given the time series [1.2, 1.3, 1.1, 8.7, 1.2, ...], identify the anomaly."

After (~60%+ expected accuracy):
"Time series statistics: length=200, mean=1.25, std=0.42, min=0.81, max=8.7,
values_at_indices_0_5=[1.2, 1.3, 1.1, 8.7, 1.2, 1.0]

Given the time series [1.2, 1.3, 1.1, 8.7, 1.2, ...]:
Step 1: Identify the global statistical baseline (mean, typical range).
Step 2: Scan for values exceeding 3 standard deviations from the mean.
Step 3: Characterize the anomaly type and likely cause.
What is the anomalous value and at which index does it occur?"
```

3. Add a visualization (line plot with the anomaly region highlighted) for multimodal models.
4. Use CoT prompting — explicitly require step-by-step reasoning about what constitutes normal behavior before identifying deviations.
5. Re-evaluate stratified by local vs. global scope to confirm improvement.

**Example 3: Creating a cross-modal time series alignment test**

User: "I need to test whether our model can match time series plots to their text descriptions."

Approach:
1. This is a cross-modal alignment task (TS→Sem and Sem→TS directions).
2. Generate candidate series from the same domain (e.g., all climate data) so distractors are plausible:

```python
# Generate 4 candidate series with distinct but overlapping characteristics
candidates = {
    'A': 'Gradual upward trend with strong 12-month seasonality, low noise',
    'B': 'Flat trend with strong 12-month seasonality, high noise',
    'C': 'Gradual upward trend with weak seasonality, low noise',
    'D': 'Sharp upward trend with strong 6-month seasonality, moderate noise'
}
```

3. Format as multiple-choice:

```
TS→Sem direction:
"Given the following time series [plot/values], which description best matches?
(A) Gradual upward trend with strong annual seasonality and low noise
(B) Flat trend with strong annual seasonality and high noise
(C) Gradual upward trend with weak seasonality and low noise
(D) Sharp upward trend with semi-annual seasonality and moderate noise"

Sem→TS direction:
"Which of the following 4 time series best matches: 'A gradually increasing
signal with pronounced 12-month cycles and minimal random variation'?"
```

4. Ensure distractors differ on anchor statistics (min/max, trend slope, periodicity) since models primarily use these cues.
5. Evaluate with exact-match accuracy. Expect strong performance (>90%) from frontier models on this task type.

## Best Practices

- **Do:** Always compute and prepend a statistical prefix (length, min, max, mean, std, boundary values, autocorrelation) when feeding time series to LLMs. This is the single highest-impact intervention, often recovering performance equivalent to adding a dedicated encoder.
- **Do:** Use chain-of-thought prompting for any temporal reasoning task. Structure the CoT as: identify features → state temporal relationships → draw conclusions. This outperforms parameter scaling from 14B to 32B.
- **Do:** Stratify evaluation by the five taxonomy dimensions and by local/global scope. Aggregate accuracy hides systematic weaknesses — seasonality detection and causal reasoning are consistently the hardest tasks.
- **Do:** Generate distractors from overlapping domains when building multiple-choice TSQA. Same-domain distractors that differ on key statistical anchors create meaningful discrimination challenges.
- **Avoid:** Relying on specialized time series encoders as the primary improvement lever. MMTS-BENCH shows encoder architecture (MLP vs. CNN vs. Transformer) and depth produce marginal differences — invest effort in backbone LLM quality and prompting strategy instead.
- **Avoid:** Single-turn question generation for real-world time series data. The three-stage pipeline (context integration → QA synthesis → verification) significantly reduces spurious cues and templated artifacts compared to direct LLM generation.

## Error Handling

- **Model returns generic or evasive answers on local tasks**: The model likely lacks sufficient local context. Provide windowed statistics (min, max, mean for the specific index range) and a zoomed plot of the relevant segment. Explicitly state the index range in the question.
- **Numerical answers are wildly off**: Check whether the statistical prefix was included. Without it, models lack the scale/offset information needed for absolute value estimation. Also verify the series wasn't silently truncated by context length limits.
- **Sequence matching returns random accuracy**: Verify that DTW distances between the correct match and distractors are sufficiently different. If all candidates have similar DTW scores, the question is ambiguous. Increase the morphological distinctiveness of distractors.
- **CoT doesn't help or hurts performance**: This occurs primarily on straightforward feature-analysis tasks (e.g., "Is the trend upward or downward?") where overthinking introduces errors. Reserve CoT for multi-step temporal reasoning, causal, and counterfactual questions.
- **Cross-modal alignment accuracy is near-random**: Ensure the text descriptions contain discriminative statistical cues (specific values, ranges, periodicities) rather than vague qualitative language. Models align primarily on quantitative anchors.

## Limitations

- The taxonomy assumes clean, well-structured time series. Highly irregular, event-driven, or sparse time series (e.g., point processes) are not well-covered by the five-dimension framework.
- Multimodal gains depend heavily on the model's vision-language fusion quality. Adding plots to a model with weak vision capabilities can degrade performance (observed with GPT-4.1 Mini on InWild tasks).
- The benchmark uses temperature 1.0 with 5-run averaging. Results at temperature 0 or with different sampling strategies may differ.
- Sequence matching via DTW works well for morphological similarity but does not capture semantic similarity (two series with different shapes but similar domain meaning).
- The benchmark evaluates understanding and reasoning, not forecasting. If the goal is to predict future values rather than answer questions about existing series, this framework addresses only the reasoning component.
- Real-world QA generation still requires human expert review (Fleiss' kappa = 0.73) — fully automated pipelines risk introducing spurious cues that inflate apparent model performance.

## Reference

**Paper**: [MMTS-BENCH: A Comprehensive Benchmark for Time Series Understanding and Reasoning](https://arxiv.org/abs/2602.08588v1) — Yin et al., KDD 2026.
**What to look for**: The hierarchical taxonomy (Section 3), the three-stage QA construction pipeline (Section 4), statistical prefix ablations (Table 7), encoder architecture ablations (Section 5.4), and the local vs. global performance gap (Section 5.2). The key actionable insight is that prompting strategy (statistical prefix + CoT) dominates over architectural choices for time series LLM performance.