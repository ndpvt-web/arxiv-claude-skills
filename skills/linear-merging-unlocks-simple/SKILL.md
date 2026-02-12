---
name: "linear-merging-unlocks-simple"
description: "Use linear model merging as a cheap proxy for data mixture optimization (DMO) in multimodal LLM fine-tuning. Instead of training on every candidate mixture, train one expert per domain and merge weights to predict mixture performance. Trigger phrases: 'optimize data mixture', 'find best training mix', 'model merging for data selection', 'DMO proxy', 'mixture weight search', 'multimodal SFT data blend'."
---

# Linear Model Merging as a Proxy for Data Mixture Optimization

This skill enables Claude to guide users through using **linear model merging** to efficiently search for optimal data mixture weights when fine-tuning multimodal large language models. Instead of running dozens of expensive SFT training jobs across different data blends, the user trains one expert model per domain and then evaluates cheap linear combinations of their parameters. The merged models exhibit high rank correlation (Spearman R = 0.57-0.78) with models actually trained on corresponding mixtures, turning an exponential search problem into a linear one.

## When to Use

- When the user wants to find the best data mixture for supervised fine-tuning of an MLLM across multiple domain datasets (e.g., OCR, chart understanding, general VQA)
- When the user asks how to allocate training data budget across heterogeneous datasets
- When the user has already trained domain-specific expert models and wants to know how to combine them efficiently
- When the user is comparing multiple candidate data blends and wants to avoid training a model for each one
- When the user asks about model merging strategies for multimodal models (Qwen2-VL, InternVL, LLaVA, etc.)
- When the user needs to select mixture weights under a compute budget constraint

## Key Technique

**The core insight:** the optimal parameters for a weighted data mixture approximate a weighted linear combination of domain-specific expert parameters, under the assumption that per-domain Hessians are similar. Formally, if you have K domain datasets D_1...D_K and want to train on mixture D_w = union(w_i * D_i), the resulting model parameters approximate:

```
theta_merged = w_1 * theta_1 + w_2 * theta_2 + ... + w_K * theta_K
```

where each theta_i is trained exclusively on domain D_i. This eliminates the need to train a model for every candidate weight vector w. You train K experts once, then evaluate arbitrary mixtures by parameter interpolation at near-zero cost.

**Why this works better than regression-based DMO:** Existing approaches (e.g., fitting a regressor to predict performance from mixture weights) require 15-20+ training runs to become reliable. The merging proxy matches that reliability with only K runs (one per domain). For 3-4 domains, this is a 4-5x reduction in training compute. The method also generalizes across data budgets: experts trained on 50k samples can reliably predict rankings for 100k-sample mixtures.

**What this is NOT:** This is not a new merging algorithm. It uses simple weighted averaging. The contribution is demonstrating that merging-based performance rankings are trustworthy proxies for actual mixture-trained performance rankings, making merging a practical tool for navigating the mixture weight search space.

## Step-by-Step Workflow

1. **Partition your training data into K domain-specific subsets.** Each domain should represent a coherent capability (e.g., document OCR, chart QA, general visual understanding, counting/perception). Use 4 or fewer domains to maintain strong rank correlation.

2. **Train one expert model per domain using LoRA SFT.** Fine-tune the same base model (e.g., Qwen2-VL-7B) on each domain independently. Use identical hyperparameters across all experts: AdamW optimizer, learning rate 2e-5 with linear warmup (10% of steps) and cosine decay, batch size 128, LoRA rank 16 applied to all LLM linear projections. Use at least 50k samples per domain (10k showed weak correlation in the paper).

3. **Define a grid of candidate mixture weight vectors.** For 2 domains, use step size 1/8 (yields 7 candidates). For 3 domains, use step size 1/8 over the simplex (yields 21 candidates). For 4+ domains, sample 20-30 points from a symmetric Dirichlet distribution. All weights must be non-negative and sum to 1.

4. **For each candidate weight vector, construct the merged model.** Compute `theta_merged = sum(w_i * theta_i)` by linearly interpolating the LoRA parameters (or full parameters) of the K experts. This is a pure arithmetic operation on tensors -- no GPU training required.

5. **Evaluate each merged model on your target benchmarks.** Run inference on all relevant evaluation sets. This is the only compute-intensive step per candidate, but inference is far cheaper than training.

6. **Rank the candidate weight vectors by merged-model performance.** The ranking of merged models reliably preserves the ranking of actual mixture-trained models (Spearman R ~ 0.70-0.78 for 2-3 domains).

7. **Select the top-ranked weight vector as your final data mixture.** Use these weights to construct your actual training dataset by sampling proportionally from each domain.

8. **Train the final production model on the selected mixture.** This single training run produces a model within 1% of the exhaustive-search optimum in the majority of benchmarks.

9. **(Optional) Cross-budget shortcut:** If full-budget expert training is too expensive, train proxy experts on 50% of the data budget, use them to identify the best mixture, then train the final model at full budget. The paper confirms this maintains strong rank correlation.

## Concrete Examples

**Example 1: Finding the best 3-domain mixture for a Qwen2-VL SFT run**

User: "I have three datasets for fine-tuning Qwen2-VL-7B: 100k OCR samples, 100k general VQA samples, and 100k chart understanding samples. How do I find the best mixture ratio without training dozens of models?"

Approach:
1. Train 3 LoRA experts on the same Qwen2-VL-7B base, one per domain (OCR, VQA, Charts), each using 100k samples with identical hyperparameters.
2. Generate 21 candidate weight vectors on the 3-simplex with step 1/8: (1,0,0), (7/8,1/8,0), (6/8,2/8,0), ..., (0,0,1).
3. For each weight vector, merge the 3 LoRA adapters: `lora_merged = w_ocr * lora_ocr + w_vqa * lora_vqa + w_chart * lora_chart`.
4. Evaluate each merged model on target benchmarks (e.g., DocVQA, ChartQA, VQAv2).
5. Rank the 21 candidates by average benchmark score.

Output:
```
Weight vector ranking (top 5 by average benchmark score):
  #1: OCR=0.375, VQA=0.375, Charts=0.250  -> avg score: 72.4
  #2: OCR=0.250, VQA=0.500, Charts=0.250  -> avg score: 72.1
  #3: OCR=0.500, VQA=0.250, Charts=0.250  -> avg score: 71.8
  ...

Recommended mixture: 37.5% OCR, 37.5% General VQA, 25% Charts
Total training runs: 3 (experts) + 1 (final) = 4 runs
Naive grid search would require: 21 runs
```

**Example 2: Budget-constrained proxy search**

User: "I need to optimize a 4-domain data mix but can only afford ~6 training runs total."

Approach:
1. Train 4 experts at 50% data budget (50k samples each instead of 100k). That's 4 runs.
2. Sample 20 candidate weight vectors from Dirichlet(1,1,1,1).
3. Merge and evaluate all 20 candidates (inference only, no training).
4. Select the top-performing weight vector.
5. Train 1 final model on the full 100k budget using the selected mixture. Total: 5 training runs.

Output:
```
Cross-budget DMO results:
  Proxy experts trained on: 50k samples/domain
  Target budget: 100k samples/domain
  Rank correlation (proxy vs full-budget): R = 0.71

  Best mixture (proxy-selected): Domain1=0.30, Domain2=0.25, Domain3=0.25, Domain4=0.20
  Final model performance vs exhaustive search optimum: -0.4% average gap
  Compute savings: 5 runs vs ~24 runs (4.8x reduction)
```

**Example 3: Implementing the merge in Python**

User: "Show me how to merge LoRA adapters with custom weights."

```python
import torch
from peft import PeftModel, PeftConfig

def merge_lora_experts(base_model, expert_paths, weights):
    """Merge K LoRA experts with specified mixture weights.

    Args:
        base_model: The base model (e.g., Qwen2-VL-7B)
        expert_paths: List of paths to LoRA adapter checkpoints
        weights: List of floats summing to 1.0, one per expert
    Returns:
        Merged LoRA state dict
    """
    assert abs(sum(weights) - 1.0) < 1e-6, "Weights must sum to 1"

    merged_state = {}
    for path, w in zip(expert_paths, weights):
        state = torch.load(f"{path}/adapter_model.bin", map_location="cpu")
        for key, param in state.items():
            if key in merged_state:
                merged_state[key] += w * param
            else:
                merged_state[key] = w * param.clone()

    return merged_state

# Usage
weights = [0.375, 0.375, 0.250]  # OCR, VQA, Charts
experts = ["./expert_ocr", "./expert_vqa", "./expert_charts"]
merged = merge_lora_experts(base_model, experts, weights)

# Save and evaluate
torch.save(merged, "./merged_adapter/adapter_model.bin")
```

## Best Practices

- **Do:** Use identical training hyperparameters, base model, and LoRA configuration for all domain experts. Inconsistencies break the linear interpolation assumption.
- **Do:** Train each expert with at least 50k samples. The paper found 10k-sample experts produced weak rank correlations, while 50k was sufficient.
- **Do:** Constrain weights to be non-negative and sum to 1. Negative weights or weights > 1 produce extrapolations that violate the proxy assumption.
- **Do:** Use Spearman rank correlation (not raw score comparison) to validate the proxy. The merged model scores are not accurate absolute predictions -- they predict the *ranking* of mixtures.
- **Avoid:** Using more than 4 domains without validating correlation first. The paper observed correlation dropping (R = 0.57 in worst 4-domain case) as domain count increased.
- **Avoid:** Comparing merged-model absolute scores to mixture-trained scores. The proxy is for ranking candidates, not predicting exact performance numbers.

## Error Handling

- **Weak rank correlation:** If the merged proxy rankings don't align with a few spot-check training runs, check that (a) all experts used the same base model checkpoint, (b) training hyperparameters were identical, and (c) the per-domain data budget was >= 50k. Hessian divergence across very different domains can break the linear assumption.
- **Degenerate weights:** If the optimizer converges to a single-domain solution (e.g., w = [1, 0, 0, 0]), the task may be dominated by one domain. Validate by checking if the single-domain expert already outperforms all mixtures on your benchmark.
- **LoRA vs full fine-tuning mismatch:** The paper used LoRA (rank 16) for all experiments. Full fine-tuning may exhibit different interpolation dynamics. If using full fine-tuning, merge only the delta from the base model: `theta_merged = theta_base + sum(w_i * (theta_i - theta_base))`.
- **Checkpoint compatibility:** All expert checkpoints must share the same parameter keys and shapes. Verify with a shape check before merging.

## Limitations

- The method has only been validated on multimodal LLMs (Qwen2-VL, InternVL). Transfer to text-only LLMs or other architectures is plausible but unverified.
- Rank correlation degrades as the number of domains increases beyond 3-4. For 8+ domains, consider hierarchical grouping into macro-domains first.
- The linear interpolation assumption relies on domain Hessians being approximately equal. Domains with very different loss landscapes (e.g., code generation vs. image captioning) may violate this.
- The proxy identifies the best *ranking* of mixtures, not the exact optimal weights. The selected mixture is typically within 1% of the true optimum, but guarantees are empirical, not theoretical.
- The approach requires all experts to be trained from the same base checkpoint. You cannot merge experts that were fine-tuned from different base models.

## Reference

**Paper:** [Linear Model Merging Unlocks Simple and Scalable Multimodal Data Mixture Optimization](https://arxiv.org/abs/2602.04937v1) (Berasi et al., 2026). Key sections: Algorithm 1 (the full pipeline), Section 4.2 (rank correlation results across 14 benchmarks), Section 4.3 (cross-budget proxy validation), and Eq. 10 (theoretical justification via second-order Taylor expansion showing when linear merging approximates the mixture optimum).