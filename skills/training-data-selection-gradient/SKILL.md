---
name: "training-data-selection-gradient"
description: "Implement Orthogonal Gradient Selection (OGS) for efficient domain adaptation of LLMs—select training data whose gradients are orthogonal to general-knowledge anchors to prevent catastrophic forgetting. Use when: 'select training data for domain fine-tuning', 'fine-tune without catastrophic forgetting', 'build a data selection pipeline for LLM adaptation', 'implement gradient-based data filtering', 'curate domain-specific training sets efficiently', 'adapt an LLM to medical/legal/finance without losing general ability'."
---

# Training Data Selection with Gradient Orthogonality (OGS)

This skill enables Claude to implement and guide users through **Orthogonal Gradient Selection (OGS)**, a data selection method for domain-adapting large language models without catastrophic forgetting. OGS uses a small Navigator model to score candidate training samples by computing how orthogonal each sample's gradient is to a general-knowledge anchor gradient. Samples whose gradients are orthogonal (neither helping nor hurting general capabilities) are preferred, allowing the target LLM to learn domain knowledge in a "safe subspace." A reinforcement learning policy (PPO-Lagrangian) dynamically balances domain learning against general capability retention across clustered data, achieving 3x data efficiency over random selection.

## When to Use

- When the user wants to **fine-tune an LLM on a specialized domain** (medical, legal, financial, scientific) and is concerned about losing general reasoning ability
- When the user asks to **select a subset of training data** that maximizes domain performance while minimizing forgetting
- When the user needs to **build a data curation pipeline** that filters domain datasets before fine-tuning
- When the user is implementing **gradient-based data selection** or influence-function-style filtering for LLM training
- When the user wants to **reduce fine-tuning compute costs** by training on fewer, higher-quality samples (e.g., 10% of available data matching 30% random performance)
- When the user asks about **preventing catastrophic forgetting** during domain adaptation without modifying the optimizer

## Key Technique

**The core insight:** Instead of modifying the optimizer at training time (gradient surgery/projection), OGS moves the geometric safety check to the data selection stage. For each candidate training sample, OGS computes an **orthogonality score**: `Orth(x_i) = 1 - |cos(g_i, g_ref)|`, where `g_i` is the sample's gradient and `g_ref` is an anchor gradient computed from a small set of general-knowledge examples (e.g., 150 GSM8K + 150 MMLU + 100 Alpaca samples). Samples scoring near 1.0 induce parameter updates that neither help nor harm general capabilities—they live in the "safe subspace" perpendicular to the general-knowledge direction.

**Navigator-Target architecture:** Computing per-sample gradients on a 70B model is prohibitive. OGS uses a small proxy model (the "Navigator," e.g., 0.5B-1B parameters) from the same model family to compute gradient geometries. The key empirical finding is that **relative gradient geometry transfers across model scales**—cosine similarities between sample gradients and anchor gradients are highly correlated between a 0.5B Navigator and a 70B Target, even though absolute magnitudes differ. The Navigator processes the full candidate pool once offline, and the learned selection policy transfers to the Target at ~0.7% computational overhead.

**RL-based dynamic selection:** Rather than using a fixed orthogonality threshold, OGS trains a PPO-Lagrangian policy that treats data selection as a constrained MDP. The state encodes gradient geometry statistics across data clusters and current model performance. The action selects which data cluster to sample from. The reward reflects domain performance improvement, while a cost signal measures general capability degradation. A Lagrange multiplier auto-tunes: when forgetting exceeds a budget, the policy shifts toward safer (more orthogonal) samples; when general performance is stable, it selects more aggressively for domain gain.

## Step-by-Step Workflow

1. **Construct the anchor dataset.** Curate 300-500 exemplars representing general capabilities to protect. Use stratified sampling: ~150 math reasoning examples (GSM8K), ~150 world knowledge examples (MMLU), ~100 instruction-following examples (Alpaca). These must cover the specific capabilities you want to preserve.

2. **Compute the anchor gradient.** Load your Navigator model (smallest model from the same family as Target, e.g., Qwen3-0.6B for Qwen3-14B Target). Run a forward-backward pass over the anchor dataset and average the gradients: `g_ref = (1/|D_anchor|) * sum(grad_L(x, theta))` for all anchor samples. Store this vector.

3. **Score every candidate domain sample.** For each sample `x_i` in your domain training pool, compute its gradient `g_i` on the Navigator, then calculate:
   - Orthogonality: `Orth(x_i) = 1 - |cos(g_i, g_ref)|` (range 0-1, higher = safer)
   - Conflict: `Conf(x_i) = -cos(g_i, g_ref)` (positive = forgetting risk, negative = synergy)

4. **Cluster the scored data.** Partition both domain and general data into K clusters (using gradient features or embedding-based clustering). Each cluster represents a coherent batch-selection option for the RL policy.

5. **Train the RL selection policy on the Navigator.** Formulate as a Constrained MDP with PPO-Lagrangian:
   - State: cluster-level orthogonality/conflict statistics + Navigator performance metrics
   - Action: select a cluster to sample the next training batch from
   - Reward: domain eval improvement
   - Cost: general eval degradation (constrained below budget epsilon)
   - Train for enough episodes that the Lagrange multiplier stabilizes

6. **Apply the learned policy to the Target model.** Use the trained policy greedily (no exploration) to select batches for the full-scale Target model. Fine-tune with standard supervised learning (LoRA recommended: rank 16, alpha 32, applied to all attention + MLP projections).

7. **Configure fine-tuning hyperparameters.** Use: learning rate 2e-4 with cosine schedule, effective batch size 64 (batch 8 x gradient accumulation 8), 3 epochs, warmup ratio 0.1, AdamW with weight decay 0.01.

8. **Evaluate on both domain and general benchmarks.** Always measure domain accuracy AND general capability (GSM8K, MMLU, ARC-C). A successful OGS run should show domain improvement with <1% general capability drop compared to the base model.

9. **Iterate on the anchor dataset if needed.** If specific general capabilities degrade, add more anchor exemplars from that category. Use active anchor selection: `D_anchor_active = argmin cos(g_D_avg, g_domain_avg)` to find anchors most vulnerable to domain-induced forgetting.

## Concrete Examples

**Example 1: Medical domain adaptation with data budget**

```
User: I have 10,000 MedQA training samples and want to fine-tune Llama-3.1-8B
for medical QA. I can only afford to train on 10% of the data. How do I pick
the best 1,000 samples without hurting the model's math ability?

Approach:
1. Load Llama-3.2-1B as Navigator (same family, 8x smaller)
2. Build anchor set: 150 GSM8K problems + 150 MMLU questions + 100 Alpaca
3. Compute g_ref on Navigator from anchor set
4. For each of the 10,000 MedQA samples, compute Orth(x_i) on Navigator
5. Rank by orthogonality score descending
6. For a quick static approach: take top 1,000 samples with Orth > 0.85
   For full OGS: cluster into K=10 groups, train PPO-Lagrangian policy,
   then let the policy dynamically select 1,000 samples during training
7. Fine-tune Llama-3.1-8B with LoRA on the selected samples

Output (expected results based on paper):
- Domain (MedQA): ~57-58% (vs ~53% random 10% selection)
- GSM8K retention: ~55% (vs ~48% with unfiltered fine-tuning)
- Training cost: 10% of full dataset, ~0.7% overhead from Navigator scoring
```

**Example 2: Implementing the scoring pipeline in Python**

```
User: Show me how to compute orthogonality scores for my training data.

Approach:
1. Write gradient computation utility
2. Compute anchor gradient
3. Score each candidate sample

Output (code):

import torch
from torch.utils.data import DataLoader
from transformers import AutoModelForCausalLM, AutoTokenizer
from peft import get_peft_model, LoraConfig

def compute_anchor_gradient(model, anchor_loader, device):
    """Average gradient over anchor dataset = general-knowledge direction."""
    model.train()
    accumulated_grad = None
    count = 0
    for batch in anchor_loader:
        batch = {k: v.to(device) for k, v in batch.items()}
        loss = model(**batch).loss
        loss.backward()
        with torch.no_grad():
            flat_grad = torch.cat([
                p.grad.flatten() for p in model.parameters() if p.grad is not None
            ])
            accumulated_grad = flat_grad if accumulated_grad is None \
                else accumulated_grad + flat_grad
            count += 1
        model.zero_grad()
    return accumulated_grad / count

def score_sample(model, sample, g_ref, device):
    """Compute orthogonality and conflict scores for one sample."""
    model.train()
    sample = {k: v.to(device) for k, v in sample.items()}
    loss = model(**sample).loss
    loss.backward()
    with torch.no_grad():
        g_i = torch.cat([
            p.grad.flatten() for p in model.parameters() if p.grad is not None
        ])
        cos_sim = torch.nn.functional.cosine_similarity(
            g_i.unsqueeze(0), g_ref.unsqueeze(0)
        ).item()
    model.zero_grad()
    orth_score = 1.0 - abs(cos_sim)   # 0-1, higher = safer
    conflict_score = -cos_sim          # positive = forgetting risk
    return orth_score, conflict_score

# Usage:
# navigator = load small model from same family as target
# g_ref = compute_anchor_gradient(navigator, anchor_loader, device)
# scores = [score_sample(navigator, s, g_ref, device) for s in domain_data]
# selected = [s for s, (orth, _) in zip(domain_data, scores) if orth > 0.85]
```

**Example 3: Diagnosing forgetting in an existing fine-tuning run**

```
User: I fine-tuned Qwen3-8B on legal data and GSM8K dropped from 72% to 58%.
Can I use gradient orthogonality to understand why?

Approach:
1. Load Qwen3-0.6B as Navigator
2. Build anchor set with 200 GSM8K problems (the degraded capability)
3. Compute g_ref from the GSM8K anchor
4. Score the legal training samples against g_ref
5. Analyze the distribution of conflict scores

Output (diagnostic):
- If many samples have Conf(x_i) > 0.3: the legal data contains samples
  whose gradients actively oppose math reasoning. These are the culprits.
  Filter them out (keep only Orth > 0.7) and retrain.
- If conflict scores are near zero but forgetting persists: the issue is
  likely training duration or learning rate, not data conflict. Reduce
  epochs from 3 to 1 or lower learning rate.
- Plot histogram of Orth scores to visualize the safety profile of your
  training data. A left-skewed distribution (many low-Orth samples)
  indicates high forgetting risk in the dataset.
```

## Best Practices

- **Do:** Use a Navigator from the **same model family** as your Target (e.g., Qwen3-0.6B for Qwen3-14B). Cross-family transfer of gradient geometry is unreliable.
- **Do:** Include **diverse general capabilities** in your anchor set—math, knowledge, instruction-following. Omitting a category leaves it unprotected.
- **Do:** Start with a **static threshold approach** (Orth > 0.8) before investing in the full RL pipeline. The static method captures most of the benefit with far less complexity.
- **Do:** Apply LoRA rather than full fine-tuning. OGS was validated with LoRA (rank 16, alpha 32) and gradient geometry is more stable in the low-rank parameter subspace.
- **Avoid:** Computing per-sample gradients directly on the Target model. The entire point of the Navigator is to make this tractable. A 0.5B Navigator scoring 10K samples costs ~15x less than doing it on an 8B model.
- **Avoid:** Using fewer than 300 anchor samples. Below this threshold, the anchor gradient becomes noisy and orthogonality scores lose discriminative power.
- **Avoid:** Setting the orthogonality threshold too aggressively (e.g., Orth > 0.95). This filters out almost all domain-relevant samples, leaving only noise-like data that teaches nothing useful.

## Error Handling

| Problem | Likely Cause | Fix |
|---|---|---|
| All samples score Orth < 0.5 | Domain data fundamentally conflicts with general knowledge | Use conflict-aware replay: mix in anchor samples during training with dynamic weighting |
| Navigator scores don't transfer to Target | Model families differ or Navigator is too small | Use a Navigator with at least 0.5B parameters from the same architecture family |
| OOM during gradient computation | Too many parameters tracked | Compute gradients only over LoRA parameters, not the full model. Use `torch.no_grad()` on frozen layers |
| Domain performance doesn't improve despite high-Orth selection | Overly conservative filtering removed informative samples | Lower the orthogonality threshold (e.g., 0.7 instead of 0.85) or switch to the RL-based dynamic policy |
| Lagrange multiplier diverges during RL training | Constraint budget epsilon is too tight | Relax epsilon (allow ~2% general degradation) or increase the dual learning rate step size |

## Limitations

- **Gradient computation cost is front-loaded.** Even with a small Navigator, computing per-sample gradients for 100K+ samples is significant. OGS is most practical when the candidate pool is 5K-50K samples.
- **Same-family requirement.** The Navigator must be from the same architecture family as the Target. You cannot use a Llama Navigator to guide Qwen fine-tuning.
- **Assumes single anchor direction.** The anchor gradient is a single averaged vector. If you need to protect many independent capabilities, the single-vector approximation may miss some. Mitigate by using multiple anchor subsets and taking the max conflict across them.
- **RL policy training adds complexity.** The full PPO-Lagrangian pipeline requires careful tuning. For most users, the static orthogonality threshold is sufficient and far simpler.
- **Validated primarily on classification/QA tasks.** The paper demonstrated results on MedQA, LegalBench, FinQA, and GSM8K. Generalization to open-ended generation tasks (creative writing, code generation) is plausible but unverified.

## Reference

**Paper:** [Training Data Selection with Gradient Orthogonality for Efficient Domain Adaptation](https://arxiv.org/abs/2602.06359v1) — Zhang et al., 2026. Focus on Section 3 (OGS method), Algorithm 1 (pseudocode), Table 1 (main results showing 3x data efficiency), and Table 2 (ablation confirming orthogonal protection is the largest contributor to forgetting prevention).