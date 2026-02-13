---
name: "weight-decay-improves-plasticity"
description: "Configure weight decay for optimal model plasticity during LLM pretraining and fine-tuning. Advise on weight decay hyperparameter selection that maximizes downstream task adaptation rather than just minimizing pretraining loss. Triggers: 'optimize weight decay for fine-tuning', 'improve model plasticity', 'pretraining hyperparameters for downstream tasks', 'weight decay schedule for LLM training', 'why does my fine-tuned model underperform', 'counterintuitive pretraining vs fine-tuning tradeoff'"
---

This skill enables Claude to advise on weight decay hyperparameter selection during LLM pretraining to maximize **plasticity** — the base model's ability to adapt to downstream tasks through fine-tuning. The core insight from Han et al. (2026) is that the weight decay value minimizing pretraining validation loss is *not* the value producing the best fine-tuned model. Higher weight decay during pretraining (often 3-10x the conventional default of 0.1) yields models that fine-tune better, even when they have slightly worse pretraining loss. This skill teaches Claude to apply that finding to real training configurations.

## When to Use

- When a user is configuring AdamW hyperparameters for LLM pretraining and plans to fine-tune afterward
- When a user observes that a model with good pretraining loss fine-tunes poorly on downstream tasks
- When a user asks how to improve fine-tuning performance without changing the fine-tuning setup itself
- When a user is running hyperparameter sweeps and needs guidance on which weight decay values to test
- When a user asks about the tradeoff between pretraining performance and downstream adaptability
- When a user wants to understand why two models with similar pretraining loss fine-tune differently
- When a user is choosing between model checkpoints and wants to predict which will fine-tune best

## Key Technique

**The plasticity-weight decay relationship.** Standard practice sets weight decay to 0.1 in AdamW and tunes it to minimize pretraining validation cross-entropy loss. This paper demonstrates that the optimal weight decay for *downstream performance after fine-tuning* is substantially higher — around 1.0 for compute-optimal training budgets (20 tokens-per-parameter) and around 0.3 for overtrained regimes (140 tokens-per-parameter). The default of 0.1 consistently underperforms for plasticity. This creates a counterintuitive scenario: a base model with *worse* pretraining loss (e.g., validation CE 2.62 vs 2.61) can produce a *better* fine-tuned model.

**Three mechanistic effects explain why higher weight decay improves plasticity.** First, it encourages **linearly separable representations** — embeddings from high-weight-decay models achieve higher linear probing accuracy on classification tasks (sentiment, topic), meaning the representation space is better organized for downstream adaptation. Second, it **regularizes attention matrices** — specifically, it halves the pseudo-rank of the query-key product matrix W_QK (from near full-rank at λ=0.1 to roughly half-rank at λ=1.0), creating lower-dimensional attention patterns that are easier for fine-tuning to reshape. Third, it **reduces the train-validation gap** monotonically, meaning less memorization of pretraining data and more generalizable features.

**Practical implication for hyperparameter selection.** If your goal is a strong fine-tuned model (not just a strong base model), you should evaluate pretraining checkpoints using plasticity-aware metrics — such as linear probe accuracy on a held-out classification task — rather than relying solely on validation cross-entropy loss. The weight decay that minimizes pretraining loss is typically too small for optimal plasticity.

## Step-by-Step Workflow

1. **Determine the training regime.** Calculate the tokens-per-parameter (TPP) ratio: divide total training tokens by model parameter count. Classify as compute-optimal (~20 TPP) or overtrained (100+ TPP). This determines the target weight decay range.

2. **Set the weight decay search range.** For compute-optimal training (~20 TPP), sweep over `{0.1, 0.5, 1.0, 1.5, 3.0}` with 1.0 as the starting point. For overtrained models (~140 TPP), sweep over `{0.1, 0.2, 0.3, 0.5, 1.0}` with 0.3 as the starting point. The pattern: longer training requires lower optimal weight decay.

3. **Keep other AdamW hyperparameters standard.** Use beta1=0.9, beta2=0.95, gradient clipping max_norm=1.0, and a warmup-cosine learning rate schedule. Weight decay in AdamW is decoupled from the gradient update (θ̂_t = θ_{t-1} - γ_t · λ · θ_{t-1}), so changing λ does not interact with the adaptive learning rate.

4. **Run pretraining with each weight decay value.** Track both validation cross-entropy loss and a plasticity proxy metric. For the proxy, periodically run linear probing on the model's last-token embeddings using a simple classification task (e.g., sentiment or topic classification with a frozen backbone and single linear head).

5. **Select the checkpoint with the highest plasticity proxy score**, not the lowest validation loss. If two checkpoints have similar probing accuracy, prefer the one with lower validation loss as a tiebreaker.

6. **Configure fine-tuning with standard settings.** Use a smaller learning rate (1e-5 for models ≤1B params, 7.5e-6 for larger), batch size 64, cosine schedule, 3 epochs, warmup ratio 0.1. The plasticity gains from pretraining weight decay are robust across these standard fine-tuning setups.

7. **Evaluate fine-tuned models with multiple metrics.** Go beyond greedy Pass@1 — use majority voting (Maj@K), best-of-K with a reward model (RM@K), Pass@K, correct ratio, and reward model scores. Plasticity improvements often show up more strongly in sampling-based metrics.

8. **When debugging poor fine-tuning results on an existing model**, check the pretraining weight decay. If it was 0.1 (the common default), the model may lack plasticity. Consider retraining with higher weight decay if feasible, or use the linear probing diagnostic to confirm low plasticity before investing in retraining.

9. **For model selection between existing checkpoints**, run linear probes on candidate base models. Higher probing accuracy on a classification task is a strong predictor of better fine-tuning performance, independent of pretraining loss.

## Concrete Examples

**Example 1: Configuring pretraining for a 1B parameter model**

User: "I'm pretraining a 1B parameter LLM on 20B tokens of FineWeb-Edu. I plan to fine-tune it on medical QA afterward. What weight decay should I use?"

Approach:
1. Calculate TPP: 20B tokens / 1B params = 20 TPP (compute-optimal regime)
2. Recommend weight decay = 1.0 as the primary value, with a sweep over {0.5, 1.0, 1.5} if budget allows
3. Note that the default 0.1 will likely yield worse fine-tuning performance despite better pretraining loss
4. Suggest tracking linear probe accuracy on a sentiment or topic task during pretraining as a plasticity indicator

Output:
```python
# Pretraining config for plasticity-optimized 1B model (20 TPP)
training_config = {
    "optimizer": "adamw",
    "learning_rate": 2e-4,
    "min_learning_rate": 2e-5,
    "weight_decay": 1.0,          # 10x the typical default of 0.1
    "beta1": 0.9,
    "beta2": 0.95,
    "max_grad_norm": 1.0,
    "batch_size": 512,
    "warmup_ratio": 0.1,
    "lr_schedule": "cosine",
    "total_tokens": 20_000_000_000,
}

# Plasticity monitoring: run every N steps
probe_config = {
    "task": "ag_news_topic_classification",  # 4-class topic probe
    "method": "linear_probe_on_last_token_embeddings",
    "freeze_backbone": True,
    "metric": "accuracy",
}
```

**Example 2: Diagnosing poor fine-tuning from an existing base model**

User: "I fine-tuned Llama-2-7B on math reasoning data but the results are disappointing. The base model's perplexity looked good. What could be wrong?"

Approach:
1. Identify that Llama-2-7B was pretrained with weight decay = 0.1 (standard Meta config)
2. Explain the plasticity gap: low weight decay yields good perplexity but potentially poor adaptability
3. Suggest a diagnostic: run linear probes on the base model's representations
4. If retraining is not feasible, suggest continued pretraining with higher weight decay as a partial remedy

Output:
```
Diagnosis: The base model likely has low plasticity due to the default weight
decay of 0.1 used during pretraining.

Diagnostic step — run a linear probe:
  1. Extract last-token embeddings from 5K examples of a classification task
  2. Train a single linear layer (frozen backbone) to predict class labels
  3. If accuracy is low relative to a model trained with higher weight decay,
     plasticity is the bottleneck

Mitigation options (ranked by effectiveness):
  A. If you can retrain: use weight_decay=1.0 for ~20 TPP budget
  B. If retraining is too expensive: run a short continued-pretraining phase
     (1-5% of original tokens) with weight_decay=1.0 before fine-tuning
  C. Adjust fine-tuning: use a higher learning rate or more epochs to
     partially compensate (less effective than fixing pretraining)
```

**Example 3: Choosing between two pretrained checkpoints**

User: "I have two checkpoints from a weight decay sweep: one with λ=0.1 (val loss 2.609) and one with λ=0.3 (val loss 2.621). Which should I fine-tune?"

Approach:
1. Note the counterintuitive finding: the higher-loss checkpoint (λ=0.3) is likely to fine-tune better
2. Recommend a quick linear probe comparison to confirm
3. Explain the mechanism: higher weight decay produces more linearly separable representations and lower-rank attention matrices, both of which facilitate adaptation

Output:
```
Choose the λ=0.3 checkpoint (val loss 2.621) for fine-tuning.

Despite its 0.012 higher pretraining loss, research shows that at overtrained
regimes, λ=0.3 consistently outperforms λ=0.1 after fine-tuning. The pretraining
loss difference is small, but the plasticity difference is substantial.

Quick validation: run linear probes on both checkpoints.
  - Extract embeddings from SST-2 (sentiment) or AG News (topic)
  - Train a linear classifier on frozen embeddings
  - The checkpoint with higher probe accuracy will fine-tune better
  - Expect λ=0.3 to show meaningfully higher probe accuracy
```

## Best Practices

- **Do** set weight decay to 1.0 for compute-optimal training (≈20 TPP) and 0.3 for overtrained models (≈140 TPP) as strong defaults when the end goal is fine-tuning.
- **Do** use linear probing accuracy as a plasticity proxy during pretraining — it is cheap to compute and highly predictive of downstream fine-tuning performance.
- **Do** evaluate fine-tuned models with sampling-based metrics (Maj@K, Pass@K) in addition to greedy decoding, as plasticity gains are often more visible under sampling.
- **Do** apply weight decay only to weight matrices, not biases or layer norms, per standard AdamW practice — this is already the default in most frameworks (HuggingFace, PyTorch).
- **Avoid** selecting base model checkpoints based solely on pretraining validation loss — this metric does not predict fine-tuning performance reliably.
- **Avoid** assuming that a weight decay sweep's optimal value for pretraining loss is also optimal for downstream tasks — these are typically different values, with plasticity favoring higher decay.
- **Avoid** using extreme weight decay (>3.0) for overtrained models — the optimal value decreases as training length increases, and excessively high values degrade both pretraining and fine-tuning.

## Error Handling

**Weight decay too high causes training instability.** If validation loss diverges or spikes during pretraining with high weight decay (e.g., λ=10.0), reduce to the next lower value in your sweep. Very high decay effectively shrinks weights toward zero too aggressively, especially in early training.

**Linear probe accuracy is flat across checkpoints.** If probing accuracy doesn't differentiate checkpoints, the classification task may be too easy or too hard. Switch to a different probe task (e.g., from binary sentiment to 4-class topic classification) or use a harder subset.

**Fine-tuning gains don't materialize despite high probe accuracy.** Ensure the fine-tuning hyperparameters are reasonable (learning rate ~1e-5, 3 epochs, cosine schedule). If the fine-tuning task is very different from the probe task domain, the probe may not be predictive for that specific downstream task.

**Framework applies weight decay to all parameters.** Verify that your training framework excludes biases and normalization layers from weight decay. In HuggingFace Transformers, this is handled by default in `AdamW`, but custom training loops may need explicit parameter group configuration.

## Limitations

- These findings are validated on models up to 4B parameters. Scaling behavior to 70B+ models is not empirically confirmed, though the mechanistic explanations (representation separability, attention rank reduction) are architecture-general.
- The optimal weight decay value depends on the TPP ratio and may need recalibration for training regimes very different from 20-140 TPP.
- Plasticity improvements are demonstrated for supervised fine-tuning (SFT). The effect on RLHF/DPO alignment stages is not studied.
- The technique optimizes the *pretrained base model* for downstream adaptation. It does not help if the fine-tuning procedure itself is poorly configured.
- Results are shown on Llama-2 and OLMo-2 architectures. Other architectures (e.g., mixture-of-experts, state-space models) may behave differently.

## Reference

**Paper:** Han, Bordt, Zhang, Kakade. "Weight Decay Improves Language Model Plasticity." arXiv:2602.11137v1, 2026. https://arxiv.org/abs/2602.11137v1

**What to look for:** Table 1 for optimal weight decay values across model sizes and TPP ratios; Figure 3 for the counterintuitive pretraining-loss vs. fine-tuning-performance tradeoff; Figure 5 for linear probe results correlating with plasticity; Section 4 for the three mechanistic effects (representation separability, attention rank, overfitting).