---
name: "multi-task-code-data-mix"
description: |
  Guide Claude to choose and configure multi-task strategies for small code LLMs — data mixing vs model merging —
  based on model scale, task correlation, and deployment constraints. Applies findings from Zhu et al. (2026).
  Trigger phrases:
  - "combine code generation and summarization into one model"
  - "should I merge my fine-tuned models or mix the training data"
  - "multi-task fine-tuning strategy for a small code LLM"
  - "deploy one model for multiple code tasks"
  - "merge LoRA adapters for code generation and documentation"
  - "data mix ratio for multi-task code training"
---

# Multi-Task Code LLM Strategy: Data Mixing vs Model Merging

This skill enables Claude to advise on and implement multi-task strategies for small code LLMs (1B–7B parameters) that need to handle both code generation and code understanding tasks (summarization, documentation, translation) within a single deployed model. The core technique is a scale-dependent decision framework: use **data mixing** (combined training data, single fine-tuning run) for models at ~2B parameters or below, and use **model merging** (separate specialist fine-tunes combined via weight-space methods like DELLA or DARE) for models at ~7B parameters or above. This is grounded in empirical weight-correlation analysis — smaller models learn highly correlated parameter updates across tasks, so joint training works well; larger models develop more independent task representations, so post-hoc merging preserves specialization better.

## When to Use

- When a user wants a single small code LLM to handle both code generation and code summarization/documentation
- When choosing between training one model on mixed data vs merging two separately fine-tuned specialist models
- When deciding merge method parameters (DELLA, DARE, TIES, or linear) for combining code task specialists
- When planning a resource-constrained deployment that cannot afford separate models per task
- When a user asks about mixing ratios or weighting for multi-task code training data
- When evaluating whether a merged or mixed model will degrade code generation Pass@1 scores
- When designing an agentic coding framework that pairs a small specialized model with a frontier model

## Key Technique

**The scale-dependent insight.** At smaller scales (~1.5B–2B parameters), fine-tuning on a shuffled combination of code generation data (e.g., KodCode, 268K examples) and code summarization data (e.g., CodeXGLUE, 417K examples) loses only ~1–2% average performance compared to individual specialists. Merging at this scale is catastrophic, causing 7–27% degradation. The reason: weight-correlation analysis shows smaller models develop highly correlated parameter updates (Pearson r > 0.5) across tasks — the tasks don't fight for model capacity, so joint training works naturally.

**At larger scales (~7B parameters), the relationship inverts.** Task-specific weight deltas become less correlated (r < 0.3 across layers), meaning each task carves out its own parameter subspace. Here, model merging methods — particularly DELLA (magnitude-based sampling with aligned parameter updates) for Qwen-family models and DARE (drop-and-rescale) for DeepSeek-family models — retain 96%+ of specialist performance and can even exceed it. The best merged Qwen2.5-Coder 7B hit 92.7% Pass@1 on HumanEval vs 90.9% for the code-generation-only specialist.

**Weight analysis as a diagnostic.** Before committing to a strategy, compute the layer-wise L2 distance between each specialist's weights and the base model, then compute Pearson correlation between the two task deltas at each layer. High cross-task correlation (> 0.4 average) predicts data mixing will work; low correlation (< 0.3) predicts merging will work better.

## Step-by-Step Workflow

1. **Identify the target model family and scale.** Determine the base model (e.g., Qwen2.5-Coder, DeepSeek-Coder, CodeLlama) and parameter count. This is the single strongest predictor of which strategy to use.

2. **Enumerate the target tasks.** Classify each task as generative (code completion, generation, repair) or comprehension (summarization, documentation, code translation). Note dataset sizes for each.

3. **Apply the scale heuristic.** For models <= 2B parameters, default to data mixing. For models >= 7B parameters, default to model merging. For models in between (3B–6B), proceed to step 4 for diagnostics.

4. **Run weight-correlation diagnostics (optional, for ambiguous cases).** Fine-tune two specialist models for ~0.5 epochs each. Compute layer-wise task deltas: `delta_task = weights_finetuned - weights_base`. Compute Pearson correlation between delta vectors at each transformer layer. Average correlation > 0.4 favors mixing; < 0.3 favors merging.

5. **If data mixing: prepare the combined dataset.** Concatenate all task datasets and shuffle uniformly. No complex ratio tuning is needed — simple concatenation with shuffling performed comparably to ratio-optimized mixes in the paper's experiments. Use the training configuration below.

6. **If model merging: fine-tune individual specialists.** Train one model per task using full fine-tuning with the hyperparameters in the reference config. Validate each specialist independently on its target benchmark before merging.

7. **Select and configure the merge method.** For Qwen-family models, use DELLA with weight ratios swept from 0.3/0.7 to 0.7/0.3 (code-gen/code-sum). For DeepSeek-family models, use DARE with similar ratio sweeps. For unknown architectures, start with linear merging at 0.5/0.5 as a baseline, then try TIES.

8. **Sweep merge weight ratios.** Test at least 5 configurations (e.g., 0.3/0.7, 0.4/0.6, 0.5/0.5, 0.6/0.4, 0.7/0.3) on a held-out validation set. Evaluate code generation with Pass@1 and summarization with BLEU-4. Pick the ratio with the best harmonic mean of normalized scores.

9. **Validate the merged/mixed model.** Run full benchmark evaluation: HumanEval and MBPP for generation, CodeXGLUE for summarization. Compare against individual specialists. Accept the multi-task model if average degradation is < 5% across tasks.

10. **Deploy and monitor.** Serve the single multi-task model. Track per-task metrics in production. If one task degrades beyond acceptable thresholds, consider asymmetric merge ratios or falling back to separate specialist deployment.

## Reference Training Configuration

```yaml
# Hyperparameters from the paper (Table 1)
small_model:  # ~1.5B-2B parameters
  learning_rate: 1e-5
  effective_batch_size: 128
  gradient_accumulation_steps: 8
  max_sequence_length: 4096
  epochs: 2
  warmup_ratio: 0.03
  lr_scheduler: cosine
  optimizer: AdamW

large_model:  # ~7B parameters
  learning_rate: 5e-6
  effective_batch_size: 64
  gradient_accumulation_steps: 8
  max_sequence_length: 4096
  epochs: 2
  warmup_ratio: 0.03
  lr_scheduler: cosine
  optimizer: AdamW

# Infrastructure: 4x H100 GPUs
# ~4 hours for 1.5B models, ~20 hours for 7B models
```

## Concrete Examples

**Example 1: Deploying a single 7B model for generation + docs**

```
User: I have a Qwen2.5-Coder 7B model. I need it to handle both code
completion and docstring generation. Should I train on combined data
or merge two fine-tuned models?

Approach:
1. Model is 7B → scale heuristic says model merging.
2. Fine-tune Specialist A on KodCode (code generation, 268K examples)
   with lr=5e-6, batch=64, 2 epochs.
3. Fine-tune Specialist B on CodeXGLUE (summarization, 417K examples)
   with identical hyperparameters.
4. Merge using DELLA (recommended for Qwen family).
5. Sweep weight ratios: test 0.5/0.5, 0.6/0.4, 0.7/0.3 (gen/sum).
6. Evaluate on HumanEval (target: >90% Pass@1) and CodeXGLUE
   (target: BLEU-4 > 0.055).

Expected outcome:
- HumanEval Pass@1: ~90.2% (specialist: 90.9%, merged retains 99.2%)
- MBPP Pass@1: ~84.9% (specialist: 87.8%)
- CodeXGLUE BLEU-4: ~0.058 (specialist: 0.064)
- Single model serves both tasks with <4% average degradation.
```

**Example 2: Resource-constrained 1.5B deployment**

```
User: I only have budget for a 1.5B parameter model. I want it to do
code generation and code summarization. What's my best strategy?

Approach:
1. Model is 1.5B → scale heuristic says data mixing.
2. Combine KodCode (268K) + CodeXGLUE (417K) into one dataset.
3. Shuffle uniformly — no ratio tuning needed.
4. Fine-tune once with lr=1e-5, batch=128, 2 epochs.
5. Evaluate on HumanEval, MBPP, CodeXGLUE.

Expected outcome:
- HumanEval Pass@1: ~76.8% (specialist: 75.6%, mixing EXCEEDS it)
- MBPP Pass@1: ~75.4% (specialist: 75.9%, only -0.7%)
- CodeXGLUE BLEU-4: ~0.048 (specialist: 0.050, only -3.6%)
- One training run, one model, minimal degradation.

Why not merging here:
- Linear merge at this scale: HumanEval drops to 67.1% (-11.2%)
- Merging causes catastrophic interference in small models.
```

**Example 3: Diagnosing an unknown 3B model**

```
User: I have a 3B code model. Not sure whether to mix or merge. How
do I decide?

Approach:
1. Scale is ambiguous (between 2B and 7B) → run diagnostics.
2. Fine-tune two specialists for 0.5 epochs each (~1 hour on 4xH100).
3. Compute weight deltas: delta_gen = W_gen_specialist - W_base,
   delta_sum = W_sum_specialist - W_base.
4. For each transformer layer j, compute:
   r_j = pearson_correlation(delta_gen_j.flatten(),
                              delta_sum_j.flatten())
5. Average r across all layers.

Decision:
- If avg(r) > 0.4 → data mixing (tasks share capacity well)
- If avg(r) < 0.3 → model merging (tasks use independent subspaces)
- If 0.3 <= avg(r) <= 0.4 → try both, pick winner on validation set
```

## Merge Method Selection Guide

```
┌─────────────────────────────────────────────────────┐
│              Model Family Decision Tree              │
├─────────────────────────────────────────────────────┤
│                                                     │
│  Qwen-family (Qwen Coder, CodeQwen)                │
│    └─ Use DELLA (best empirical results)            │
│       └─ Start with weight ratio 0.5/0.5            │
│                                                     │
│  DeepSeek-family (DeepSeek Coder)                   │
│    └─ Use DARE (best empirical results)             │
│       └─ Start with weight ratio 0.5/0.5            │
│                                                     │
│  Unknown / Other architecture                       │
│    └─ Start with Linear merge (0.5/0.5 baseline)   │
│       └─ If insufficient, try TIES                  │
│          └─ If still insufficient, try DARE         │
│                                                     │
└─────────────────────────────────────────────────────┘
```

## Best Practices

- **Do** check model scale first — it is the strongest single predictor of which strategy works. The 2B/7B boundary is well-established across two model families.
- **Do** sweep merge weight ratios in increments of 0.1. The optimal ratio varies by model family and task balance. Don't assume 0.5/0.5 is best.
- **Do** validate on both tasks simultaneously. A merge that preserves 99% of code generation but loses 30% of summarization is not a successful multi-task model.
- **Do** use the weight-correlation diagnostic when model scale falls in an ambiguous range (3B–5B) or when working with a model family not covered by the paper.
- **Avoid** complex data mixing ratios for the mixing strategy. Simple concatenation + shuffle matched or beat ratio-tuned mixes. Don't over-engineer the data pipeline.
- **Avoid** merging models below 2B parameters. The paper shows 7–27% degradation — the capacity is too constrained for weight-space combination to preserve both tasks.
- **Avoid** assuming LoRA merging behaves identically to full-weight merging. The paper used full fine-tuning. If using LoRA adapters, expect different optimal ratios and potentially different strategy preferences.

## Error Handling

- **Merge produces gibberish or mode collapse.** The merge ratio is too extreme (e.g., 0.9/0.1). Reset to 0.5/0.5 and verify both specialist models independently produce valid outputs before re-merging.
- **Code generation Pass@1 drops sharply after merging.** The generation task is being diluted. Increase the code-generation weight (e.g., 0.7/0.3) and try TIES which resolves sign conflicts between task deltas.
- **Data-mixed model is worse than either specialist.** Check for data contamination or format conflicts between datasets. Ensure both datasets use consistent prompt templates and tokenization. Also verify the combined dataset is properly shuffled.
- **Weight correlation analysis gives inconsistent results across layers.** Focus on middle layers (layers 8–24 in a 32-layer model). Early and final layers often show task-agnostic patterns. If middle layers disagree, lean toward the strategy that matches the model's closest scale bracket.
- **Training diverges on mixed data.** The learning rate may be too high for the combined data distribution. Reduce by 2x and increase warmup ratio to 0.05.

## Limitations

- **Only validated on code generation + summarization.** The paper does not test other task combinations (e.g., code repair + test generation + vulnerability detection). Extrapolate to other task pairs with caution.
- **Full fine-tuning only.** All experiments used full parameter fine-tuning, not LoRA/QLoRA. The scale heuristic and merge method recommendations may not transfer directly to adapter-based fine-tuning.
- **Two model families tested.** Results are demonstrated on Qwen Coder and DeepSeek Coder. Other architectures (CodeLlama, StarCoder, Phi) may have different optimal strategies.
- **Two scales tested.** The 1.5B vs 7B comparison leaves a gap in the 3B–5B range where the optimal strategy is uncertain.
- **English-centric benchmarks.** HumanEval, MBPP, and CodeXGLUE are English-dominant. Multi-task strategies for multilingual code tasks are not evaluated.
- **Two-task limit.** The paper merges exactly two specialist models. Scaling to 3+ tasks introduces additional interference that may require hierarchical merging strategies.

## Reference

**Paper:** Zhu, M., Sobolev, B., Krishna, R., Pavuluri, R., & Patterson, S. (2026). *Multi-task Code LLMs: Data Mix or Model Merge?* arXiv:2601.21115v1. [https://arxiv.org/abs/2601.21115v1](https://arxiv.org/abs/2601.21115v1)

**What to look for:** Table 3 for full benchmark comparison across all methods and scales; Table 1 for training hyperparameters; Figures 4–5 for weight-correlation analysis and the L2-distance/Pearson-correlation diagnostic; Section 5 for the practical decision framework.