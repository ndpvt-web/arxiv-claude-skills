---
name: "bagging-based-merging-robust-general"
description: |
  Implement BOOM (Bagging-based rObust mOdel Merging) to create robust general-purpose
  text embedding models by training on sampled data subsets and merging via Multi-SLERP
  or Task Arithmetic. Supports incremental learning without full retraining.
  Trigger phrases:
  - "merge embedding models for better generalization"
  - "train robust text embeddings with bagging"
  - "incrementally update my embedding model with new data"
  - "improve out-of-domain embedding performance"
  - "combine multiple fine-tuned embedding models"
  - "set up a BOOM training pipeline"
---

# BOOM: Bagging-Based Model Merging for Robust Text Embeddings

This skill enables Claude to guide users through implementing the BOOM (Bagging-based rObust mOdel Merging) pipeline for training and merging text embedding models. BOOM trains multiple embedding models on independently sampled data subsets, then merges their parameters into a single model using spherical interpolation (Multi-SLERP) or task-vector methods. The result is a single embedding model with stronger in-domain accuracy, better out-of-domain generalization, and native support for incremental updates — all at single-model inference cost.

## When to Use

- When the user wants to train a general-purpose text embedding model on multi-task data and needs robust OOD generalization (e.g., training on retrieval/STS/classification corpora but deploying to unseen code or domain-specific retrieval).
- When the user needs to incrementally add new embedding training data (a new domain, language, or task type) without retraining from scratch on the full corpus.
- When the user has multiple fine-tuned embedding checkpoints (e.g., one per task or data source) and wants to merge them into a single performant model.
- When the user observes that their batch-level-shuffled embedding model overfits to in-domain benchmarks but degrades on held-out domains.
- When the user asks how to apply bagging or ensemble-style techniques at the parameter level for embedding models.
- When the user is building a sentence-transformer or dense retrieval pipeline and wants to improve robustness without increasing inference cost.

## Key Technique

**Why bagging at the model level works.** Standard multi-task embedding training shuffles all datasets at the batch level (BLS). While BLS yields the strongest average in-domain scores, it has two weaknesses: (1) it can overfit to the training distribution, hurting out-of-domain generalization, and (2) any new data requires a full retrain. BOOM addresses both by borrowing the bagging principle from ensemble learning — but instead of ensembling predictions, it merges model *parameters*. Each sub-model sees a different random sample of the training data, so the merged model captures a more diverse region of the loss landscape, acting as an implicit regularizer that improves OOD robustness.

**How merging works.** Given M sub-models with weights W₁..Wₘ trained from the same base model, BOOM merges them using Multi-SLERP (spherical linear interpolation generalized to M vectors). Each weight tensor is decomposed into magnitude and direction, then directions are interpolated on the hypersphere while magnitudes are linearly combined. The formula is: `W_merged = (Σαᵢ||Wᵢ||) · exp_M(Σαᵢ log_M(Wᵢ/||Wᵢ||))`, where αᵢ are weights proportional to subset size or set to equal. Alternative methods (Task Arithmetic, TIES-Merging) perform comparably, but Multi-SLERP is the recommended default.

**Incremental learning.** To absorb new data D' without retraining on the full corpus, BOOM trains a lightweight update model on `D' ∪ D_core` where D_core is a small (40%) random sample of the original training data that preserves existing knowledge. The update model is then merged into the existing model: `W_dynamic = Merge(W_existing, W_update)`. This achieves performance on par with full retraining at a fraction of the compute.

## Step-by-Step Workflow

1. **Select a base model and training framework.** Start from a pretrained language model (e.g., Qwen3-0.6B for prototyping, Qwen3-4B for production). Set up sentence-transformers or a custom contrastive training loop with InfoNCE loss. For models above 1B parameters, use LoRA (rank=32, alpha=32) instead of full fine-tuning.

2. **Prepare and catalog your multi-task training data.** Organize datasets by task type (retrieval, STS, classification, clustering). Record the size of each dataset. Ensure each dataset has query-positive pairs and ideally 7 hard negatives per example.

3. **Define subset sampling ratios.** Choose M subsets with sampling ratios. Recommended configurations by compute budget:
   - **Budget-neutral (1.0x cost):** `{50}-and-R` — train one model on a random 50%, another on the remaining 50%.
   - **Moderate (1.8x cost):** `{60, 60, 60}` — three models, each on 60% of data (sampled independently per dataset).
   - **Best quality (3.0x cost):** `{20, 40, 60, 80, 100}` — five models with increasing coverage.
   - **Recommended sweet spot:** `{80, 100}` — two models, strong performance at 1.8x cost.

4. **Sample subsets independently per dataset.** For each sub-model m and each dataset Dᵢ, sample kₘ% of Dᵢ *without replacement*. This ensures each sub-model has proportional representation from every task. Do NOT sample at the corpus level (which would skew toward large datasets).

5. **Train each sub-model independently.** Train each sub-model on its sampled subset using identical hyperparameters: lr=5e-5, batch_size=256 (or 128 for 4B), max_seq_len=512, epochs=1. Each training run uses batch-level shuffling within its subset. All sub-models share the same base checkpoint initialization.

6. **Merge sub-models using Multi-SLERP.** After all sub-models finish training, merge their weight tensors using Multi-SLERP. Set merge weights αᵢ proportional to subset sizes (size-weighted) or equal (both work similarly). Use a library like `mergekit` or implement SLERP manually for each parameter tensor.

7. **Evaluate on in-domain and OOD benchmarks.** Test the merged model on your target benchmarks (e.g., MTEB for English, CMTEB for Chinese). Compare against a single-model BLS baseline. BOOM should match or exceed in-domain scores and show clear OOD gains.

8. **For incremental updates: sample a core subset.** When new training data arrives, randomly sample 40% of the original training data as D_core. This is a one-time operation — cache D_core for reuse across multiple incremental updates.

9. **Train the update model on new + core data.** Fine-tune a fresh copy of the base model on `D_new ∪ D_core` using the same training hyperparameters as step 5. This is far cheaper than retraining on the full corpus.

10. **Merge the update model into the existing model.** Apply Multi-SLERP to merge `W_existing` and `W_update` with equal weights. The result is a single model that incorporates the new knowledge without catastrophic forgetting of existing capabilities.

## Concrete Examples

**Example 1: Building a robust multi-task embedding model**

```
User: I have 15 retrieval, STS, and classification datasets totaling 2M examples.
I want to train a single embedding model that generalizes well to unseen domains.

Approach:
1. Use Qwen3-0.6B as base model with full-parameter fine-tuning.
2. Define two subsets: {80, 100} — one model on 80% of each dataset,
   one on 100%.
3. For each of the 15 datasets, independently sample 80% for subset 1.
   Subset 2 uses all data.
4. Train both models with InfoNCE loss, lr=5e-5, batch=256,
   max_seq_len=512, 1 epoch, 7 hard negatives.
5. Merge with Multi-SLERP, equal weights (α₁=0.5, α₂=0.5):

   import torch
   def multi_slerp(weights, alphas):
       """Merge M weight tensors via spherical interpolation."""
       norms = [w.norm() for w in weights]
       dirs = [w / n for w, n in zip(weights, norms)]
       merged_norm = sum(a * n for a, n in zip(alphas, norms))
       # Approximate SLERP for M > 2: iterative pairwise
       merged_dir = dirs[0]
       cumulative_alpha = alphas[0]
       for i in range(1, len(dirs)):
           t = alphas[i] / (cumulative_alpha + alphas[i])
           dot = (merged_dir * dirs[i]).sum().clamp(-1, 1)
           omega = torch.acos(dot)
           if omega.abs() < 1e-6:
               merged_dir = (1 - t) * merged_dir + t * dirs[i]
           else:
               merged_dir = (torch.sin((1 - t) * omega) * merged_dir +
                             torch.sin(t * omega) * dirs[i]) / torch.sin(omega)
           cumulative_alpha += alphas[i]
       return merged_norm * merged_dir

6. Evaluate on MTEB; expect +0.3-0.5% mean score and stronger OOD retrieval.

Output: A single merged checkpoint with same architecture and inference
cost as either sub-model, but better generalization.
```

**Example 2: Incrementally adding a new domain**

```
User: My production embedding model was trained on general English data.
I now have 200K new medical retrieval pairs. I don't want to retrain
from scratch on the full 2M + 200K corpus.

Approach:
1. Sample D_core: randomly select 40% of the original 2M training examples
   (800K), preserving per-dataset ratios.
2. Combine: D_update = D_medical(200K) + D_core(800K) = 1M total.
3. Train W_update: fine-tune a fresh copy of the original base model
   (NOT the production model) on D_update with same hyperparameters.
4. Merge: W_production_v2 = Multi-SLERP(W_production, W_update, α=[0.5, 0.5]).
5. Evaluate on both original benchmarks and medical retrieval test set.

Output: Updated model handles medical queries while retaining general
performance. Training cost: ~50% of full retraining (1M vs 2.2M examples).
```

**Example 3: Merging task-specific embedding checkpoints**

```
User: I have three separately fine-tuned checkpoints: one for retrieval,
one for STS, and one for classification. Can I merge them?

Approach:
1. Verify all three were fine-tuned from the SAME base model.
   (Merging divergent architectures fails.)
2. Load all three checkpoints.
3. Merge using Multi-SLERP with equal weights (α=1/3 each):

   from mergekit.config import MergeConfiguration
   # Or manual merge per parameter tensor:
   for name in param_names:
       merged[name] = multi_slerp(
           [ckpt1[name], ckpt2[name], ckpt3[name]],
           [1/3, 1/3, 1/3]
       )

4. Evaluate on all three task types. If one task degrades significantly,
   try increasing its weight (e.g., α=[0.2, 0.4, 0.4]).

Output: Single model competitive across all three tasks. If task conflicts
are low (typical for embedding tasks), expect minimal degradation vs.
individual specialists.
```

## Best Practices

- **Do:** Sample subsets independently per dataset, not globally across the corpus. Global sampling skews representation toward large datasets.
- **Do:** Initialize all sub-models from the same base checkpoint. Merging requires a shared parameter space; divergent initializations produce incoherent merges.
- **Do:** Use size-weighted or equal merge weights as a starting point. The paper finds both work well; tune only if evaluation shows clear imbalance.
- **Do:** Cache a 40% core subset for incremental updates. Drawing a new core each time introduces unnecessary variance.
- **Avoid:** Merging models trained for different numbers of epochs or with very different learning rates. Parameter drift between sub-models should be comparable in magnitude for effective SLERP.
- **Avoid:** Using BOOM with fewer than 2 sub-models or with extremely small sampling ratios (e.g., 10%). Sub-models trained on too little data produce weak checkpoints that drag down the merge.
- **Avoid:** Applying BOOM when datasets have strong inter-task conflicts (rare for embeddings, but possible). Check: if sequential single-task training badly hurts prior tasks, conflicts exist and simple merging may not resolve them.

## Error Handling

- **Merge produces NaN or degenerate weights:** This occurs when weight tensors are near-zero or near-parallel, causing division-by-zero in SLERP. Add a small epsilon (1e-8) to norms and fall back to linear interpolation when the angle omega < 1e-6.
- **OOD performance doesn't improve over BLS:** Ensure subsets are genuinely different. If all subsets use >90% overlap, they converge to the same solution. Use ratios with meaningful variation (e.g., {50, 100} not {95, 100}).
- **Incremental update degrades existing tasks:** The core subset may be too small. Increase from 40% to 60%. Alternatively, reduce the merge weight of the update model (e.g., α_update=0.3).
- **Sub-model training diverges:** For large models (4B+), ensure LoRA is used. Full fine-tuning on small subsets can overfit rapidly with high learning rates.
- **Checkpoint sizes mismatch:** All sub-models must have identical architecture. If using LoRA, merge LoRA weights back into the base before applying Multi-SLERP.

## Limitations

- BOOM requires training M separate models, so total training compute is 1.0x-3.0x of a single run (though these parallelize across GPUs). The method saves nothing when a single training run is already the bottleneck.
- The technique is designed for embedding models trained with contrastive losses (InfoNCE). Applicability to generative language models or other architectures is not established.
- Merging assumes limited inter-task conflict. For task combinations with strong negative transfer (e.g., conflicting label semantics across classification datasets), merging may not resolve the underlying conflict.
- Multi-SLERP operates in parameter space, which requires all sub-models to share the same architecture and initialization. You cannot merge models of different sizes or architectures.
- The incremental learning protocol assumes the distribution of original training data is well-represented by a 40% sample. Highly imbalanced or long-tailed datasets may need stratified core sampling.

## Reference

[Bagging-Based Model Merging for Robust General Text Embeddings](https://arxiv.org/abs/2602.05787v2) — Zhang et al., 2026. Focus on Section 4 (BOOM method), Table 3 (sampling ratio ablations), and Section 5 (incremental learning protocol with core subset construction).