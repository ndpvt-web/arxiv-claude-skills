---
name: "rethinking-llm-as-a-judge-representation-as-a-judg"
description: "Build probing-based evaluation pipelines that judge LLM output quality using hidden-state representations from small language models instead of expensive LLM-as-a-Judge prompting. Implements the INSPECTOR framework for aspect-level scoring. Triggers: 'evaluate LLM outputs cheaply', 'build an evaluation pipeline with probing', 'replace LLM-as-a-Judge with representations', 'score reasoning quality using hidden states', 'filter training data with representation probes', 'INSPECTOR framework evaluation'"
---

# Representation-as-a-Judge: Probing-Based LLM Output Evaluation

This skill enables Claude to design and implement evaluation pipelines that judge LLM-generated text using internal hidden-state representations from small language models (0.6B-1.7B parameters) rather than expensive prompted LLM-as-a-Judge calls. Based on the INSPECTOR framework, the approach extracts hidden states from small models processing (prompt, response) pairs, trains lightweight linear probes on those representations, and predicts aspect-level quality scores (semantic consistency, logicality, informativeness, fluency, factuality) — achieving 80-90% binary F1 while being orders of magnitude cheaper than prompting GPT-4 or DeepSeek-V3.

## When to Use

- When the user needs to evaluate thousands or millions of LLM-generated responses and LLM-as-a-Judge API costs are prohibitive
- When building a data curation or filtering pipeline for supervised fine-tuning datasets
- When the user wants to score reasoning quality (math, logic, QA) across multiple aspects without autoregressive decoding
- When designing a scalable evaluation system that must run on limited GPU resources (a single consumer GPU suffices for small models)
- When the user asks to replace prompted evaluation with something faster, cheaper, or more reproducible
- When building automated quality gates in CI/CD for LLM output pipelines
- When the user needs interpretable evaluation — understanding which layers and features drive quality judgments

## Key Technique

**The Semantic Capacity Asymmetry Hypothesis** is the core insight: evaluation requires significantly less semantic capacity than generation. A 0.6B-parameter model that produces mediocre text when prompted still encodes rich evaluative signals in its hidden states. This asymmetry means you can use a tiny model as a "sensor" — feeding it the text to evaluate and reading its internal representations — without ever asking it to generate an evaluation in words.

**INSPECTOR** operationalizes this insight through a three-stage pipeline. First, a strong judge model (e.g., DeepSeek-V3 or GPT-4) annotates a calibration set of (prompt, response) pairs with aspect-level scores on a 1-5 scale across five dimensions: semantic consistency, logicality, informativeness, fluency, and factuality. Second, the same (prompt, response) pairs are fed through a small model (Qwen3-0.6B, Qwen3-1.7B, Llama-3.2-1B), and per-layer hidden states are extracted using multiple pooling strategies (mean, last-token, min, max, min-max-mean concatenation). Third, lightweight logistic regression classifiers are trained on PCA-reduced hidden-state features plus statistical summaries (norm, variance, entropy), selecting the best layer-pooling combinations via cross-validation. The resulting probes run inference in milliseconds with no autoregressive decoding.

**Why this works in practice**: mid-to-upper layers of small transformers capture abstract quality signals that correlate strongly with expert judgments. Binary classification (quality threshold >= 4 on a 5-point scale) achieves 80-90% F1, vastly outperforming the same small model prompted to evaluate (20-40% F1). For data filtering, probed scores produce SFT models comparable to those trained on data filtered by full LLM judges.

## Step-by-Step Workflow

1. **Define evaluation aspects and task scope.** Choose which quality dimensions matter for your use case. The default five (semantic consistency, logicality, informativeness, fluency, factuality) cover reasoning tasks well. For code evaluation, substitute dimensions like correctness, efficiency, and security.

2. **Collect a calibration dataset.** Assemble 500-2000 (prompt, response) pairs spanning the full quality range. Use a medium-sized model (e.g., Llama-3-8B-Instruct) at varied temperatures (0.3-1.0) to generate responses with natural quality diversity across a given benchmark.

3. **Annotate with a strong judge.** Score each response on each aspect (1-5 scale) using a frontier LLM (GPT-4, DeepSeek-V3, Claude). Use structured output with explicit rubrics per aspect. This is a one-time cost — typically a few hundred API calls.

4. **Balance the label distribution.** Determine the minimum sample count across all five score levels per aspect. Randomly downsample majority classes to match. For binary classification, apply threshold tau=4 (scores >= 4 are high-quality).

5. **Extract hidden-state representations.** Load a small model (Qwen3-0.6B or Qwen3-1.7B recommended). For each (prompt, response) pair, run a forward pass and cache hidden states from all layers. Apply five pooling strategies per layer: mean across tokens, last token, min, max, and concatenation of min/max/mean.

6. **Engineer features from hidden states.** For each layer-pooling combination, compute: (a) PCA-reduced embedding (d=50), (b) vector norm, (c) element-wise variance, (d) Shannon entropy of the softmax-normalized vector, (e) attention entropy from that layer's attention weights. Concatenate into a feature vector.

7. **Select optimal layers via greedy search.** Run 5-fold stratified cross-validation with logistic regression (C in [0.001, 0.01, 0.1, 1], macro F1 scoring) for each layer-pooling-feature configuration. Rank by F1. Pick the top-K layers, then greedily add layers that improve combined performance. Mid-to-upper layers (60-80% depth) typically dominate.

8. **Train the final probe classifiers.** For each aspect, train a logistic regression classifier on the selected layer features using StandardScaler normalization. Evaluate on a held-out test split. Store the trained sklearn pipeline (scaler + PCA + classifier) as a serialized artifact.

9. **Deploy as a scoring service.** At inference time, feed new (prompt, response) pairs through the small model, extract hidden states from the selected layers only, compute features, and run through the trained probes. Each evaluation takes milliseconds with no autoregressive decoding. Return per-aspect scores and an aggregate quality ranking.

10. **Validate against held-out judge labels periodically.** Monitor probe accuracy on a rolling sample annotated by the original strong judge. Retrain probes if the input distribution shifts significantly (new tasks, different response styles).

## Concrete Examples

**Example 1: Building a math reasoning evaluator**

User: "I have 50K math problem solutions generated by different models. I need to filter for high-quality reasoning before fine-tuning, but GPT-4 evaluation would cost too much."

Approach:
1. Sample 1,500 diverse (problem, solution) pairs from the 50K pool, ensuring coverage across difficulty levels.
2. Score the sample using DeepSeek-V3 on five aspects: semantic consistency, logicality, informativeness, fluency, factuality (1-5 scale each). Cost: ~$5-15 one-time.
3. Load Qwen3-1.7B locally. Extract hidden states for all 1,500 pairs across all 24 layers with 5 pooling strategies.
4. Train logistic regression probes per aspect. Greedy layer selection identifies layers 16, 19, 21 as most informative.
5. Run the trained probes on all 50K solutions. Filter to responses where all aspects score >= 4 (binary threshold).

Output:
```python
# Probe training results (per-aspect binary F1):
# semantic_consistency: 0.87
# logicality: 0.84
# informativeness: 0.82
# fluency: 0.91
# factuality: 0.85

# Filtering results:
# Total responses: 50,000
# High-quality (all aspects >= 4): 18,342 (36.7%)
# Processing time: ~12 minutes on single A100
# Equivalent GPT-4 cost avoided: ~$800
```

**Example 2: CI quality gate for a chatbot pipeline**

User: "We deploy a customer-support chatbot. I want an automated quality check that flags low-quality responses before they reach users."

Approach:
1. Collect 800 historical (customer_query, bot_response) pairs with human quality ratings.
2. Map human ratings to the five evaluation aspects. Annotate any gaps with Claude as the strong judge.
3. Load Llama-3.2-1B on the serving infrastructure (fits in 2GB VRAM).
4. Extract hidden states and train probes. Optimize for high recall on the "low-quality" class to minimize missed failures.
5. Deploy the probe as a sidecar service. Each response is scored in <50ms. Responses below threshold are routed to human review.

Output:
```python
# Deployed probe configuration:
{
    "model": "Llama-3.2-1B",
    "selected_layers": [8, 11, 13],  # mid-to-upper of 16 total
    "pooling": "mean",
    "classifier": "logistic_regression",
    "threshold": 0.65,  # tuned for 95% recall on low-quality
    "latency_p99_ms": 38,
    "memory_mb": 2100
}
# Quality gate results (on test set):
# Precision on flagged responses: 0.72
# Recall on true low-quality: 0.95
# Responses requiring human review: 11% of traffic
```

**Example 3: Comparing evaluation strategies for a research benchmark**

User: "I'm running GPQA evaluations. Should I use LLM-as-a-Judge or try representation probing?"

Approach:
1. Run both approaches on a shared 500-sample GPQA subset.
2. LLM-as-a-Judge: prompt Qwen3-1.7B directly to evaluate responses. Measure F1 and cost.
3. INSPECTOR probing: extract Qwen3-1.7B hidden states, train probes using 400 samples, test on 100.
4. Compare against DeepSeek-V3 gold labels.

Output:
```
| Method                        | Binary F1 | Cost (500 samples) | Latency/sample |
|-------------------------------|-----------|--------------------:|---------------:|
| DeepSeek-V3 prompted (gold)   | 1.00      |             ~$2.50  |        ~1.5s   |
| Qwen3-1.7B prompted           | 0.31      |              $0.00  |        ~0.8s   |
| Qwen3-1.7B INSPECTOR probe    | 0.86      |              $0.00  |       ~0.02s   |

Verdict: Probing the same small model that fails at prompted evaluation
recovers most of the signal from the frontier judge at 75x lower latency
and zero marginal API cost.
```

## Best Practices

- **Do:** Use at least 500 balanced calibration samples per aspect. Probe accuracy degrades sharply below 300 samples due to class imbalance in multiclass settings.
- **Do:** Focus on binary classification (high/low quality) when practical. Binary probes achieve 80-90% F1 vs. 50-60% for 5-class, and binary decisions suffice for most filtering and gating tasks.
- **Do:** Prefer mid-to-upper layers (60-80% of model depth) as the starting search range. These consistently carry the strongest evaluative signals across model families.
- **Do:** Combine multiple pooling strategies in your feature vectors. Mean pooling captures global semantics; last-token pooling captures conclusion signals; min/max capture outlier activations.
- **Avoid:** Probing only the final layer. The last layer is optimized for next-token prediction and often carries weaker evaluative signal than layers 2-4 below it.
- **Avoid:** Using the same model for both response generation and evaluation probing. The probe may learn to detect stylistic artifacts rather than quality.
- **Avoid:** Skipping PCA reduction. Raw hidden states (d=2048+) cause overfitting in logistic regression. Reducing to d=50 consistently improves generalization.

## Error Handling

- **Class imbalance in calibration data**: If most responses cluster at one quality level, the probe will be biased. Downsample majority classes before training. If a score level has fewer than 30 samples, merge it with an adjacent level.
- **Layer selection instability**: If cross-validation scores are noisy across runs, increase folds from 5 to 10 and use repeated stratified CV. If instability persists, fix layers to the 65-75% depth range empirically.
- **Distribution shift at inference**: If the probe encounters response styles very different from calibration data (e.g., trained on English, deployed on code), F1 will degrade silently. Maintain a rolling validation set scored by the strong judge and trigger retraining when probe agreement drops below 75%.
- **Memory issues with large datasets**: Extract and save hidden states to disk (HDF5 or memory-mapped numpy arrays) rather than holding all layers in GPU memory. Only the selected 2-4 layers need to be loaded for inference.
- **Scikit-learn pipeline serialization**: Save the full pipeline (StandardScaler + PCA + LogisticRegression) via `joblib.dump`. Version the pipeline alongside the small model checkpoint to prevent feature-dimension mismatches.

## Limitations

- **Calibration requires a strong judge**: The probe is only as good as its training labels. You still need an initial investment in frontier-model annotations for the calibration set. This approach reduces ongoing cost, not setup cost.
- **Domain specificity**: Probes trained on math reasoning do not transfer well to creative writing or code evaluation. Retrain for each domain.
- **Multiclass granularity is limited**: Fine-grained 5-class scoring reaches only 50-60% F1. If you need precise numeric scores (not just high/low), LLM-as-a-Judge remains more accurate.
- **Model-family dependency**: Probes are tied to a specific model architecture and checkpoint. Switching from Qwen3-1.7B to Llama-3.2-1B requires full retraining — the hidden-state geometry differs.
- **No explanations**: Unlike prompted judges, probes output scores without rationales. If you need justification for why a response scored low, you still need generative evaluation for those flagged cases.
- **Not validated beyond reasoning tasks**: The paper demonstrates results on GSM8K, MATH, and GPQA. Performance on open-ended generation, summarization, or dialogue quality is not yet established.

## Reference

[Rethinking LLM-as-a-Judge: Representation-as-a-Judge with Small Language Models via Semantic Capacity Asymmetry](https://arxiv.org/abs/2601.22588v1) — Li et al., 2026. Focus on Section 3 (INSPECTOR framework architecture), Section 4 (layer selection and feature engineering), and Table 2 (binary/multiclass F1 results across model sizes) for implementation details.