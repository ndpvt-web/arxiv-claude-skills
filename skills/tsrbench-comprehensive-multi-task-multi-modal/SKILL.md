---
name: "tsrbench-comprehensive-multi-task-multi-modal"
description: "Evaluate and build multi-modal time series reasoning pipelines using the TSRBench framework. Covers perception, reasoning, prediction, and decision-making over time series data represented as text, plots, or both. Use when: 'evaluate LLM on time series tasks', 'build a time series reasoning benchmark', 'test model on anomaly detection and forecasting', 'analyze time series with multiple modalities', 'compare text vs visual time series performance', 'create time series QA evaluation'."
---

# TSRBench: Multi-Task Multi-Modal Time Series Reasoning

This skill enables Claude to build, run, and interpret comprehensive time series reasoning evaluations using the TSRBench methodology. TSRBench organizes time series reasoning into four dimensions — Perception, Reasoning, Prediction, and Decision-Making — across 15 distinct tasks spanning 14 real-world domains. The core insight is that time series can be represented as numerical text sequences or matplotlib plots, and these modalities are complementary but current models fail to fuse them effectively. This skill teaches Claude to construct evaluation pipelines, format time series for multi-modal input, design task-specific prompts, and interpret results through the lens of TSRBench's findings on scaling laws, reasoning-prediction decoupling, and modality fusion.

## When to Use

- When the user wants to evaluate an LLM or VLM on time series understanding tasks (anomaly detection, pattern analysis, causal reasoning, forecasting)
- When building a benchmark or test suite that probes a model's ability to reason over temporal data
- When designing prompts that present time series as text, images, or both to a generalist model
- When the user asks to compare how well different models handle numerical prediction vs. semantic reasoning on time series
- When constructing multi-modal evaluation pipelines that feed time series as both numerical sequences and plots
- When analyzing whether a model's strong reasoning ability translates to accurate forecasting (it often does not)
- When the user wants to build domain-specific time series QA (finance, healthcare/ECG, energy, traffic, climate)

## Key Technique

**Multi-Modal Time Series Representation.** TSRBench's central contribution is treating time series as a first-class evaluation modality for generalist models. Each time series problem can be presented in three ways: (T) as a textual numerical sequence embedded in a prompt, (V) as a matplotlib-rendered line plot at 100 PPI resolution, or (T+V) both simultaneously. The benchmark found that text and visual representations achieve similar overall accuracy but are "strongly complementary" — they succeed on different subsets of problems. Despite this, current multi-modal models fail to exploit the complementarity when given both inputs together.

**Four-Dimensional Task Taxonomy.** Problems are organized into Perception (pattern analysis, noise understanding, anomaly detection, similarity analysis), Reasoning (etiological, causal, abductive, temporal, numerical, deductive, inductive), Prediction (forecasting, event prediction), and Decision-Making (qualitative, quantitative). This hierarchy moves from basic pattern extraction through logical inference to actionable outputs. A critical finding is that scaling laws hold for perception and reasoning (bigger models do better) but break down for prediction — GPT-4-class models do not reliably outperform smaller ones at numerical forecasting.

**Reasoning-Prediction Decoupling.** The benchmark demonstrates that a model's ability to reason about time series semantically (e.g., identifying trends, explaining causes) does not predict its ability to forecast numerical values. This means evaluation pipelines must test both capabilities independently, and systems that need forecasting should not assume a strong reasoner will also be a strong predictor. Tool augmentation (extracting statistical features like mean, variance, change points via scipy) provides only marginal gains (+1.2% max), suggesting the bottleneck is in the model's numerical reasoning rather than feature access.

## Step-by-Step Workflow

1. **Define the evaluation scope.** Select which of the 4 dimensions and 15 tasks to evaluate. Map user requirements to specific tasks: anomaly detection falls under Perception, causal analysis under Reasoning, forecasting under Prediction, and portfolio/control decisions under Decision-Making.

2. **Prepare time series data in dual modalities.** For each problem, create both a textual representation (comma-separated numerical values with timestamps) and a visual representation (matplotlib line plot rendered at 100 PPI). Use consistent formatting:
   ```python
   # Textual format
   text_repr = "Timestamps: [t0, t1, ..., tn]\nValues: [v0, v1, ..., vn]"

   # Visual format
   import matplotlib.pyplot as plt
   fig, ax = plt.subplots(figsize=(6, 4), dpi=100)
   ax.plot(timestamps, values)
   ax.set_xlabel("Time"); ax.set_ylabel("Value")
   fig.savefig("ts_plot.png", dpi=100)
   ```

3. **Construct domain-contextualized prompts.** Frame each question with domain-specific expertise. Use role-based system prompts (e.g., "You are a cardiologist analyzing ECG data" for healthcare, "You are a quantitative analyst" for finance). Include the time series data, any contextual event descriptions, and a multiple-choice answer format.

4. **Design answer format as multiple-choice.** TSRBench uses multiple-choice for all tasks including forecasting (to reduce difficulty and enable clean accuracy measurement). Provide 4-5 options where distractors are plausible but verifiably incorrect. For numerical tasks, space distractors at meaningful intervals.

5. **Run inference across modality conditions.** Evaluate each problem under three conditions: text-only (T), vision-only (V), and combined (T+V). For API-based models, use the appropriate endpoint. For open-source models, use vLLM with the corresponding input format.

6. **Extract and verify answers.** Parse model outputs for the selected multiple-choice option. Use regex matching for answer extraction (e.g., match `(A)`, `A.`, `Answer: A` patterns). Flag responses that don't clearly select an option for manual review.

7. **Compute per-task and per-dimension accuracy.** Calculate accuracy = correct / total for each of the 15 tasks, then aggregate by dimension. Compare T vs V vs T+V performance to identify modality-specific strengths and fusion effectiveness.

8. **Analyze scaling and decoupling effects.** Compute Spearman rank correlation between model size and per-dimension accuracy. Check whether perception/reasoning correlate with scale (they should) and whether prediction does (it likely won't). Cross-correlate reasoning accuracy with prediction accuracy to test for decoupling.

9. **Generate diagnostic report.** Produce a structured summary showing: overall accuracy, per-dimension breakdown, modality comparison (T vs V vs T+V), tasks where the model is weakest, and whether the model exhibits reasoning-prediction decoupling.

10. **Iterate on prompt design or tool augmentation.** If prediction performance is low, try augmenting prompts with extracted statistical features (mean, std, trend slope, change points via `scipy.signal.find_peaks` and `numpy.polyfit`). Expect marginal gains — the benchmark shows +1.2% maximum from tool augmentation.

## Concrete Examples

**Example 1: Building an anomaly detection evaluation for an LLM**

User: "I want to test how well GPT-4 detects anomalies in time series data across different domains."

Approach:
1. Prepare 50-100 time series samples with known anomalies from domains like finance (price spikes), healthcare (ECG irregularities), and energy (consumption outliers).
2. For each sample, generate text and plot representations.
3. Construct prompts following TSRBench's Perception > Anomaly Detection template:

```python
prompt_template = """You are a {domain_expert} analyzing {domain} data.

The following time series represents {description}:
{time_series_text}

Which of the following best describes the anomaly present in this data?
(A) {option_a}
(B) {option_b}
(C) {option_c}
(D) {option_d}

Answer with the letter only."""
```

4. Run under T, V, and T+V conditions.
5. Report accuracy per domain and modality.

Output:
```
Anomaly Detection Results (GPT-4):
  Text-only:    72.3% overall (Finance: 78%, Healthcare: 65%, Energy: 74%)
  Vision-only:  69.1% overall (Finance: 64%, Healthcare: 76%, Energy: 67%)
  Combined T+V: 71.8% overall (Finance: 73%, Healthcare: 71%, Energy: 71%)

  Note: Combined mode does NOT outperform best single modality per-domain,
  consistent with TSRBench finding on multimodal fusion failure.
```

**Example 2: Testing reasoning-prediction decoupling**

User: "I want to check if my fine-tuned model that's good at causal reasoning is also good at forecasting."

Approach:
1. Select tasks from both Reasoning (Causal Discovery, Numerical Reasoning) and Prediction (Forecasting) dimensions.
2. Prepare matched problem pairs: same time series domain, one requiring causal analysis and one requiring numerical prediction.
3. Evaluate the model on both task sets independently.
4. Compute Spearman correlation between reasoning and prediction scores.

```python
from scipy.stats import spearmanr

reasoning_scores = [model_accuracy_per_series_reasoning]
prediction_scores = [model_accuracy_per_series_prediction]
rho, p_value = spearmanr(reasoning_scores, prediction_scores)

print(f"Spearman rho: {rho:.3f}, p-value: {p_value:.4f}")
# Expected: weak correlation (rho < 0.3), confirming decoupling
```

Output:
```
Reasoning accuracy:  81.2%
Prediction accuracy: 54.7%
Spearman rho: 0.18, p-value: 0.12

Diagnosis: Your model exhibits reasoning-prediction decoupling.
Strong semantic understanding does NOT transfer to numerical forecasting.
Consider dedicated forecasting heads or hybrid architectures.
```

**Example 3: Designing a multi-domain time series QA benchmark**

User: "Help me create a small benchmark to evaluate our internal model on time series reasoning across energy and traffic domains."

Approach:
1. Define task coverage: select 6 tasks (Pattern Analysis, Anomaly Detection, Causal Discovery, Numerical Reasoning, Forecasting, Qualitative Decision-Making).
2. Source or synthesize 20 problems per task per domain (240 total).
3. For synthetic data, use controlled generation:

```python
import numpy as np

def generate_energy_series(pattern="seasonal", anomaly=False, length=168):
    """Generate hourly energy consumption with known properties."""
    t = np.arange(length)
    base = 50 + 10 * np.sin(2 * np.pi * t / 24)  # daily cycle
    if pattern == "trend":
        base += 0.1 * t
    noise = np.random.normal(0, 2, length)
    series = base + noise
    if anomaly:
        idx = np.random.randint(length // 4, 3 * length // 4)
        series[idx] *= 3  # spike anomaly
    return t, series, {"pattern": pattern, "anomaly_idx": idx if anomaly else None}
```

4. Generate both textual and visual representations for each problem.
5. Write answer verification code that re-executes generation logic to confirm ground truth.
6. Package as a JSON dataset:

```json
{
  "id": "energy_anomaly_001",
  "domain": "energy",
  "dimension": "perception",
  "task": "anomaly_detection",
  "text_input": "Hourly consumption (kWh): [48.2, 51.1, ..., 152.7, ..., 49.8]",
  "image_path": "plots/energy_anomaly_001.png",
  "question": "What type of anomaly is present in this energy consumption data?",
  "options": {"A": "Point spike at hour 87", "B": "Gradual drift upward", "C": "Level shift at hour 40", "D": "No anomaly present"},
  "answer": "A",
  "metadata": {"generation_seed": 42, "anomaly_idx": 87}
}
```

## Best Practices

- **Do** always test text and vision modalities separately before combined — TSRBench shows T+V rarely beats the best single modality, so you need the baseline to diagnose fusion issues.
- **Do** use multiple-choice format for forecasting tasks to enable clean accuracy measurement; open-ended numerical prediction is much harder to evaluate reliably.
- **Do** render plots at 100 PPI resolution — TSRBench's ablation shows this balances token efficiency with visual feature visibility. Lower resolution loses detail; higher resolution wastes tokens without accuracy gains.
- **Do** include domain context in prompts (role framing, event descriptions) — the benchmark shows context-aware evaluation is essential for prediction tasks.
- **Avoid** assuming that a model good at reasoning will be good at forecasting — the decoupling finding means you must test prediction independently.
- **Avoid** relying on tool augmentation (statistical feature extraction) as a primary strategy — the benchmark found only +1.2% gain from feeding models pre-computed statistics like mean, variance, and change points.
- **Avoid** evaluating only on one domain — TSRBench spans 14 domains because performance varies dramatically across domains for the same model.

## Error Handling

- **Model refuses to answer or outputs free-form text instead of selecting an option:** Add explicit instruction reinforcement ("You must respond with exactly one letter: A, B, C, or D") and implement fallback regex patterns to extract answers from verbose responses.
- **Plot rendering fails or produces unreadable charts:** Validate that the time series length is appropriate for the plot size. Series longer than 1000 points should be downsampled or split into panels. Always include axis labels and titles.
- **Ground truth verification fails for synthetic data:** Re-run the generation function with the stored seed to reproduce exact values. If floating-point drift causes mismatches, use `np.isclose` with appropriate tolerances.
- **Model performance is near random (25% on 4-choice):** Check for prompt formatting issues — some models are sensitive to option ordering. Try shuffling option order across instances to control for position bias.
- **Combined T+V performance drops below single-modality:** This is expected behavior per TSRBench findings. Document it as a modality fusion failure rather than treating it as a bug.

## Limitations

- TSRBench uses multiple-choice format which caps the difficulty ceiling — real-world forecasting requires open-ended numerical output, which is substantially harder.
- The benchmark's finding that scaling laws break down for prediction was observed on models up to GPT-4-class; future architectures may change this.
- Synthetic data with controlled properties enables clean evaluation but may not capture the messiness of real-world time series (missing values, irregular sampling, multi-variate dependencies).
- The 100 PPI resolution finding is specific to the plot styles and series lengths in TSRBench; other visualization approaches (heatmaps, sparklines, multi-panel) may have different optimal resolutions.
- The benchmark primarily evaluates English-language prompts; cross-lingual time series reasoning is not covered.

## Reference

**Paper:** [TSRBench: A Comprehensive Multi-task Multi-modal Time Series Reasoning Benchmark for Generalist Models](https://arxiv.org/abs/2601.18744v1) — Look for: the four-dimensional task taxonomy (Table 1), modality comparison results (Table 2-3), scaling law analysis (Figure 4), and reasoning-prediction correlation analysis (Figure 5). Code and data: [github.com/tianyi-lab/TSRBench](https://github.com/tianyi-lab/TSRBench), [HuggingFace: umd-zhou-lab/TSRBench](https://huggingface.co/datasets/umd-zhou-lab/TSRBench).