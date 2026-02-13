---
name: "fit-defying-catastrophic-forgetting"
description: "Implement the FIT framework for continual LLM unlearning — removing private, copyrighted, or harmful knowledge from language models across sequential deletion requests without catastrophic forgetting. Use when: 'unlearn data from my model', 'remove private information from LLM', 'continual machine unlearning', 'handle sequential deletion requests', 'prevent catastrophic forgetting during unlearning', 'make my model forget specific knowledge'."
---

# FIT: Continual LLM Unlearning Without Catastrophic Forgetting

This skill enables Claude to guide users through implementing the FIT (Filtering, Importance-aware updates, Targeted layer attribution) framework for continual LLM unlearning. FIT processes sequential deletion requests — removing personal information, copyrighted content, or harmful material from a trained LLM — while preserving the model's general utility. Unlike one-shot unlearning methods that degrade after a handful of requests, FIT maintains stable performance across hundreds of sequential deletions by combining redundancy filtering, gradient-based algorithm selection, and selective layer updates.

## When to Use

- When the user needs to remove specific knowledge (PII, copyrighted text, harmful content) from a fine-tuned or pretrained LLM
- When deletion requests arrive sequentially over time and the model must handle many (tens to hundreds) without retraining from scratch
- When prior unlearning attempts caused catastrophic forgetting or severe utility degradation on benchmarks like MMLU, CommonsenseQA, or GSM8K
- When the user wants their unlearned model to resist recovery attacks (fine-tuning relearning, quantization-based reactivation)
- When building a compliance pipeline that must process GDPR/CCPA-style right-to-deletion requests at scale
- When evaluating unlearning quality and needing symmetric metrics that jointly assess forgetting effectiveness and utility retention

## Key Technique

**The core problem:** Standard unlearning methods (gradient ascent, negative preference optimization) work for isolated requests but degrade rapidly under continual use. After 50-100 sequential deletions, model utility collapses — the model "forgets" not just the target data but general reasoning ability. This is catastrophic forgetting in the unlearning context.

**FIT's three-pronged solution:** (1) **Data Filtering** removes redundant content from incoming deletion requests using SimCSE embedding similarity (threshold `tau`) followed by a loss-difference test (threshold `epsilon`) that preserves genuinely informative samples even if they appear similar to prior requests. This prevents the optimizer from repeatedly perturbing the same parameter regions. (2) **Importance-aware algorithm selection** computes an importance score `IMP = ||grad_{E(D_f)} L(D_f)||_2` — the L2 norm of the gradient with respect to the request's embedding — and bins it into low/medium/high tiers. Low-IMP requests get aggressive methods (RLabel), medium-IMP get balanced methods (NPO), and high-IMP get conservative methods (NPO+KL) that protect utility. This adaptive dispatch prevents over-erasing easy requests or under-erasing deeply memorized ones. (3) **Targeted layer attribution** masks each layer individually, measures the loss deviation `s_l = |L_masked - L_original|`, ranks layers by this score, and restricts gradient updates to the top-K layers (K=8 empirically optimal). Only MLP blocks and attention modules in those layers are modified; everything else stays frozen.

**Why it works:** By filtering redundant requests, choosing the right unlearning strength per request, and restricting updates to the most relevant layers, FIT avoids the diffuse parameter perturbation that causes catastrophic forgetting. Experiments on Yi-6B, Llama-2-7B, Llama-3-8B, and Llama-3-8B-Instruct show FIT handles 300 sequential deletions while maintaining the best trade-off between Forget Degree and Retain Utility, outperforming nine baselines on downstream tasks.

## Step-by-Step Workflow

1. **Prepare the forget and retain datasets.** Structure each deletion request `D_f^(t)` as a set of (prompt, completion) pairs representing knowledge to remove. Assemble a retain set `D_r` of examples the model must still answer correctly. If the original pretraining corpus is unavailable, fine-tune the base model on `D_f ∪ D_r` to create the fine-tuned model, and on `D_r` alone to create a retain-model reference.

2. **Implement redundancy filtering.** For each incoming request `D_f^(t+1)`, chunk the text into fixed-size segments. Compute SimCSE embeddings for each chunk and compare against all previously processed chunks using cosine similarity. Flag chunks with similarity above threshold `tau` (start with `tau=0.85`). For flagged chunks, compute the loss-difference: `delta_L = |L_with - L_without|` where `L_with` is cross-entropy on the full request and `L_without` excludes that chunk. Retain the chunk if `delta_L > epsilon` (start with `epsilon=0.01`). Discard truly redundant chunks to produce the filtered request `D_f^(t+1)*`.

3. **Compute the importance score.** Run a forward-backward pass on the filtered request through the current model. Calculate `IMP = ||grad_{E(D_f^(t+1)*)} L||_2` — the L2 norm of the gradient of the loss with respect to the embedding layer output. This quantifies how deeply the model has memorized the target content.

4. **Select the unlearning algorithm based on IMP tier.** Discretize IMP into three bins using percentile thresholds from your calibration set: low-IMP (below 33rd percentile) maps to RLabel (random label assignment — aggressive forgetting), medium-IMP (33rd-66th percentile) maps to NPO (negative preference optimization — balanced), high-IMP (above 66th percentile) maps to NPO+KL (negative preference optimization with KL-divergence penalty against a reference model — conservative, utility-preserving).

5. **Perform targeted layer attribution.** For each layer `l` in the model, temporarily mask it (zero out its outputs) and compute the loss on `D_f^(t+1)*`. Calculate the attribution score `s_l = |L_masked^(l) - L_original|`. Rank all layers by `s_l` in descending order. Select the top K=8 layers as the update target.

6. **Apply the selected unlearning method to the target layers only.** Freeze all parameters outside the top-K layers' MLP blocks and attention modules. Run the chosen algorithm (RLabel, NPO, or NPO+KL) for a small number of gradient steps (1-3 epochs, low learning rate ~1e-5) on the filtered request. Update only the unfrozen parameters.

7. **Evaluate using Forget Degree and Retain Utility.** Compute three metrics on both forget and retain sets: token probability, ROUGE-L, and token-level accuracy. Aggregate each triplet via geometric mean: `F = (Prob * ROUGE * Acc)^(1/3)`. Normalize against the retain model: `F.D. = max(0, 1 - |F/F_Q - 1|)` and `R.U. = max(0, 1 - |R/R_Q - 1|)`. F.D. near 1.0 means effective forgetting; R.U. near 1.0 means preserved utility.

8. **Run downstream benchmarks.** After every N deletion rounds (e.g., every 50), evaluate on MMLU, CommonsenseQA, and GSM8K to confirm general capabilities remain intact. Track the F.D./R.U. trade-off curve over time.

9. **Test recovery resistance.** Fine-tune the unlearned model on retain-only or unrelated data for several epochs and recheck F.D. — it should remain high. Quantize the model to int4 and recheck — F.D. should not drop significantly. If recovery is too easy, increase K or use a more aggressive algorithm tier.

10. **Iterate for the next deletion request.** Accept the next request `D_f^(t+2)`, repeat from step 2. The cumulative forget set grows but filtering prevents redundant accumulation, and targeted updates prevent parameter drift.

## Concrete Examples

**Example 1: Removing personal information from a fine-tuned Llama model**

User: "I fine-tuned Llama-3-8B on customer support data and need to remove 50 customers' personal information sequentially as deletion requests come in. How do I do this without destroying the model's helpfulness?"

Approach:
1. Format each customer's data as (prompt, completion) pairs — e.g., prompt: "What is John Doe's email?", completion: "john.doe@example.com"
2. Assemble a retain set from the remaining customer support conversations (without PII)
3. Create a retain-model reference by fine-tuning base Llama-3-8B on the retain set only
4. For each of the 50 deletion requests:
   - Filter redundant chunks (customers with similar data patterns get deduplicated)
   - Compute IMP — simple factual recall typically scores low-to-medium IMP
   - Select RLabel or NPO accordingly
   - Attribute layers — PII recall often concentrates in middle-to-upper MLP layers
   - Apply unlearning to top-8 layers only
5. After all 50 requests, evaluate F.D. on PII prompts (should be >0.9) and R.U. on general support quality (should be >0.85)

Output:
```
Round 10/50: F.D.=0.94, R.U.=0.91, MMLU=62.1 (baseline: 63.0)
Round 30/50: F.D.=0.92, R.U.=0.89, MMLU=61.8
Round 50/50: F.D.=0.91, R.U.=0.87, MMLU=61.5
Recovery test (fine-tune 3 epochs on retain data): F.D.=0.88 (resistant)
```

**Example 2: Removing copyrighted content from a model trained on mixed data**

User: "Our model memorized passages from 200 copyrighted books. We're getting takedown requests one at a time. I need a pipeline that won't break the model after processing all 200."

Approach:
1. Extract memorized passages per book using prompt-completion probing (prompt with first sentence, check if model completes verbatim)
2. Structure each book's data as a deletion request with 5-20 passage pairs
3. Implement the full FIT pipeline with `tau=0.85` for filtering (many literary passages share stylistic similarity)
4. Copyright passages tend to score medium-to-high IMP (deeply memorized) — expect NPO and NPO+KL to dominate algorithm selection
5. Process requests sequentially, checkpointing every 25 requests
6. Evaluate ROUGE-L between model output and original passages (should drop below 0.1 for forgotten books)

Output:
```python
# Pseudocode for the processing loop
for t, deletion_request in enumerate(book_requests):
    filtered = redundancy_filter(deletion_request, history, tau=0.85, eps=0.01)
    imp = compute_importance(model, filtered)
    algo = select_algorithm(imp)  # RLabel | NPO | NPO+KL
    target_layers = attribute_layers(model, filtered, K=8)
    model = apply_unlearning(model, filtered, algo, target_layers, lr=1e-5, epochs=2)
    if t % 25 == 0:
        evaluate(model, forget_set, retain_set, benchmarks=["MMLU", "CSQA", "GSM8K"])
        save_checkpoint(model, f"checkpoint_{t}")
```

**Example 3: Building the evaluation pipeline with F.D. and R.U. metrics**

User: "I already have an unlearning method. I just need the symmetric evaluation metrics from the FIT paper to properly measure forgetting vs utility."

Approach:
1. Collect predictions from the unlearned model on both forget and retain evaluation sets
2. Compute per-set metrics: token probability (mean log-prob of target tokens), ROUGE-L (between generated and reference), token accuracy (exact match rate at token level)
3. Compute the same three metrics from the retain model (the gold reference)
4. Aggregate and normalize

Output:
```python
import numpy as np

def compute_fd_ru(unlearned_metrics, retain_model_metrics):
    """
    unlearned_metrics: dict with keys 'forget' and 'retain',
        each containing 'prob', 'rouge', 'accuracy' floats
    retain_model_metrics: same structure for the retain-model reference
    """
    # Geometric mean aggregation
    F = np.power(
        unlearned_metrics['forget']['prob'] *
        unlearned_metrics['forget']['rouge'] *
        unlearned_metrics['forget']['accuracy'], 1/3)
    R = np.power(
        unlearned_metrics['retain']['prob'] *
        unlearned_metrics['retain']['rouge'] *
        unlearned_metrics['retain']['accuracy'], 1/3)
    FQ = np.power(
        retain_model_metrics['forget']['prob'] *
        retain_model_metrics['forget']['rouge'] *
        retain_model_metrics['forget']['accuracy'], 1/3)
    RQ = np.power(
        retain_model_metrics['retain']['prob'] *
        retain_model_metrics['retain']['rouge'] *
        retain_model_metrics['retain']['accuracy'], 1/3)

    # Symmetric normalized metrics
    forget_degree = max(0.0, 1.0 - abs(F / FQ - 1.0))
    retain_utility = max(0.0, 1.0 - abs(R / RQ - 1.0))
    return forget_degree, retain_utility
```

## Best Practices

- **Do:** Start with K=8 target layers. The paper's ablation across four models consistently found 6-9 layers optimal. K=8 balances forgetting strength with recovery resistance.
- **Do:** Calibrate IMP percentile thresholds on a held-out set of 20-30 deletion requests before deploying the pipeline. The boundaries between low/medium/high IMP vary by model and domain.
- **Do:** Always run the loss-difference test (`delta_L > epsilon`) before discarding filtered chunks. Pure embedding similarity misses cases where semantically similar text encodes distinct sensitive information.
- **Do:** Checkpoint the model and filter history every N requests so you can resume without reprocessing.
- **Avoid:** Using the same unlearning algorithm for all requests. The whole point of importance-aware selection is that aggressive methods on high-IMP data destroy utility, while conservative methods on low-IMP data leave residual memorization.
- **Avoid:** Updating all layers or using LoRA-style rank-1 updates. Full-layer updates cause catastrophic forgetting; overly restricted updates (like LoRA on 1-2 layers) leave parameters that attackers can reactivate via quantization.
- **Avoid:** Setting `tau` too low (e.g., <0.7) for the similarity filter — this discards too aggressively and can remove genuinely distinct requests that share surface-level vocabulary.

## Error Handling

- **F.D. drops sharply after many rounds:** The filtering threshold `tau` may be too aggressive, causing informative data to be discarded. Lower `tau` by 0.05 increments and reprocess recent requests.
- **R.U. degrades below 0.8:** Too many high-IMP requests are being routed to aggressive algorithms. Review the IMP percentile boundaries — the medium/high threshold may need raising.
- **Layer attribution returns inconsistent layers across similar requests:** This is expected. Different instances of the same category (e.g., different people's PII) may activate different layers. Trust the per-request attribution rather than caching layer selections.
- **Recovery attack succeeds (F.D. drops >0.15 after fine-tuning):** Increase K from 8 to 10-12 to broaden the parameter footprint of unlearning. Alternatively, use a more aggressive algorithm tier for the affected requests.
- **Out-of-memory during layer attribution:** The masking sweep requires one forward pass per layer. For very large models, batch the attribution across layers or use gradient-based approximations (`s_l ≈ ||grad_l L||`) instead of masking.

## Limitations

- FIT is designed for decoder-only transformer LLMs. Encoder-only or encoder-decoder architectures require adapting the layer attribution and embedding gradient computation.
- The method assumes access to model weights and the ability to fine-tune. It does not apply to API-only models where you cannot modify parameters.
- Redundancy filtering relies on SimCSE embeddings, which may miss semantic similarity in highly domain-specific or multilingual text. Consider domain-adapted embedding models for specialized corpora.
- The IMP-to-algorithm mapping (three tiers, three methods) was validated on the PCH benchmark. Novel domains or unusual data distributions may require recalibrating the tier boundaries and possibly adding alternative algorithms.
- Evaluation requires a retain-model reference. If you cannot construct one (no retain set available), the F.D./R.U. metrics cannot be computed as designed — fall back to absolute metric thresholds instead.
- The method does not guarantee cryptographic-level data deletion. Determined adversaries with white-box access may still extract fragments. FIT provides practical-level unlearning, not formal privacy guarantees.

## Reference

[FIT: Defying Catastrophic Forgetting in Continual LLM Unlearning](https://arxiv.org/abs/2601.21682v1) — Xu et al., 2026. Focus on Section 3 (the FIT framework components), Section 4 (PCH benchmark construction and evaluation metrics), and Section 5.3 (ablation studies on K, tau, and algorithm selection).