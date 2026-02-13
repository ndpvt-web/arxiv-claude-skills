---
name: "bear-beam-search-aware-optimization-recommendation"
description: "Implement BEAR (Beam-SEarch-Aware Regularization) to fix training-inference mismatch in LLM-based recommendation systems. Adds a token-level ranking regularizer to SFT so positive items survive beam search pruning. Use when: 'add beam search aware training to my LLM recommender', 'fix beam search dropping good recommendations', 'implement BEAR regularization for recommendation', 'optimize LLM fine-tuning for beam search inference', 'reduce incorrect pruning in generative recommendation', 'add top-B token constraint to SFT loss'."
---

# BEAR: Beam-Search-Aware Optimization for LLM Recommendation

This skill enables Claude to implement the BEAR regularization technique for LLM-based recommendation systems. Standard supervised fine-tuning (SFT) optimizes the overall probability of positive items, but beam search can still prune them if any single token in the item's identifier has insufficient prefix probability. BEAR adds a lightweight regularization term that forces every token in a positive item to rank within the top-B candidates at each decoding step, closing this training-inference gap with ~12.5% average improvement and negligible computational overhead.

## When to Use

- When building or fine-tuning an LLM-based recommendation system that uses beam search at inference time (e.g., BIGRec, LLaRA, A-LLMRec architectures)
- When debugging why a fine-tuned LLM recommender has high overall item probability but poor retrieval metrics (HitRatio, NDCG)
- When positive items are being pruned during beam search despite having reasonable total sequence probability
- When implementing constrained beam search over an item vocabulary and seeing high pruning rates (>50%)
- When adapting any seq2seq recommendation pipeline from greedy/sampling decoding to beam search and seeing performance drops
- When adding a regularization term to an existing SFT training loop for generative recommendation

## Key Technique

**The Problem:** In LLM-based recommendation, items are represented as token sequences (e.g., item titles or IDs). SFT trains the model to maximize `P(y|x) = prod P(y_t | y_{<t}, x)` — the joint probability of the full item sequence given user history. However, beam search with beam width B keeps only the top-B prefixes at each step. A positive item with high total probability can be eliminated at step t if its token `y_t` ranks outside the top-B at that step. Empirically, this "incorrect pruning" accounts for ~70% of retrieval failures.

**The Solution:** BEAR adds a regularization term that enforces a relaxed necessary condition: at every decoding step t, the ground-truth token `y_t` must rank within the top-B tokens by probability. The raw indicator function `I(P(y_t) >= beta_t^B)` (where `beta_t^B` is the B-th highest token probability) is non-differentiable, so BEAR replaces it with a sigmoid surrogate over the log-probability gap: `L_reg = sum_t log sigma_xi(log beta_t^B - log P(y_t))`. The final objective is `L_BEAR = L_SFT + lambda * L_reg`, where lambda controls regularization strength and xi controls sigmoid sharpness.

**Why It Works Efficiently:** Computing `beta_t^B` requires only a top-B selection over the softmax logits already computed during the forward pass — no additional forward passes, no beam search simulation. This adds negligible cost (~1% overhead) compared to SFT alone, while achieving a 24.86% average reduction in incorrect pruning rate.

## Step-by-Step Workflow

1. **Set up the SFT baseline.** Implement standard next-token prediction loss over item sequences: `L_SFT(x,y) = -sum_t log P_theta(y_t | y_{<t}, x)`. Confirm your model produces logits over the full vocabulary at each decoding position.

2. **Define the beam width B.** Match the beam width used at inference time (typically B=10 for recommendation). This value directly controls what "top-B" means in the regularization term.

3. **Compute the B-th highest probability at each step.** After the softmax over logits at position t, use `torch.topk(probs, B)` to extract the B-th largest probability value `beta_t^B`. This is the pruning threshold — any token below this would be eliminated by beam search.

4. **Compute the pruning margin.** Calculate `Delta_t^B = log(beta_t^B) - log(P(y_t))` for each position t in the positive item sequence. When `Delta_t^B > 0`, the positive token is outside the top-B and would be pruned.

5. **Apply the sigmoid surrogate.** Compute `L_reg = sum_t log(sigmoid(Delta_t^B / xi))` where xi is the temperature parameter (start with xi=1.0). This is differentiable and approximates the indicator function penalty.

6. **Combine with SFT loss.** Set the final training objective to `L_BEAR = L_SFT + lambda * L_reg`. Start with lambda=1.0 and tune on a validation set. Higher lambda means stronger enforcement of the top-B constraint.

7. **Apply to your training loop.** Replace the loss computation in your existing SFT training loop. No changes are needed to the data pipeline, model architecture, or optimizer. Use LoRA (rank 8) with AdamW (lr=1e-4) as a starting point.

8. **Validate with pruning rate analysis.** After training, measure the pruning rate: for each positive item in the validation set, check what percentage of items have at least one token that falls outside the top-B at any decoding step. BEAR should reduce this substantially (target: 20-30% reduction vs. SFT).

9. **Run constrained beam search at inference.** Use beam search with width B over the valid item token set. Constrain the search to only produce valid item identifiers from your catalog.

10. **Evaluate retrieval metrics.** Measure HitRatio@K and NDCG@K (K in {5, 10}) to confirm improvement over the SFT-only baseline.

## Concrete Examples

**Example 1: Adding BEAR to an existing PyTorch SFT training loop**

User: "I have an LLM recommender fine-tuned with SFT but beam search keeps missing relevant items. Add BEAR regularization."

Approach:
1. Identify the loss computation in the training loop
2. Add the BEAR regularization term alongside existing SFT loss
3. Keep all other training infrastructure unchanged

```python
import torch
import torch.nn.functional as F

def bear_loss(logits, target_ids, beam_width=10, lambda_reg=1.0, xi=1.0):
    """
    BEAR training objective combining SFT with beam-search-aware regularization.

    Args:
        logits: Model output logits, shape (batch, seq_len, vocab_size)
        target_ids: Ground-truth token IDs, shape (batch, seq_len)
        beam_width: Beam width B used at inference (default: 10)
        lambda_reg: Regularization weight lambda (default: 1.0)
        xi: Sigmoid temperature (default: 1.0)

    Returns:
        Combined BEAR loss scalar
    """
    # Standard SFT: cross-entropy over target tokens
    log_probs = F.log_softmax(logits, dim=-1)  # (batch, seq_len, vocab)
    sft_loss = F.cross_entropy(
        logits.view(-1, logits.size(-1)),
        target_ids.view(-1),
        reduction='mean'
    )

    # BEAR regularization: enforce top-B ranking at each step
    probs = F.softmax(logits, dim=-1)  # (batch, seq_len, vocab)

    # Get B-th highest probability at each position
    topk_probs, _ = torch.topk(probs, beam_width, dim=-1)  # (batch, seq_len, B)
    beta_B = topk_probs[:, :, -1]  # (batch, seq_len) — the B-th value

    # Get probability of the target token at each position
    target_probs = probs.gather(-1, target_ids.unsqueeze(-1)).squeeze(-1)

    # Pruning margin in log space
    log_beta = torch.log(beta_B + 1e-10)
    log_target = torch.log(target_probs + 1e-10)
    delta = (log_beta - log_target) / xi

    # Sigmoid surrogate penalty
    reg_loss = torch.log(torch.sigmoid(delta) + 1e-10).mean()

    return sft_loss + lambda_reg * reg_loss
```

**Example 2: Diagnosing pruning rate before and after BEAR**

User: "How do I measure whether beam search is incorrectly pruning my positive items?"

Approach:
1. Run teacher-forced forward pass on validation data
2. At each decoding step, check if the target token ranks within top-B
3. Report per-item and aggregate pruning rates

```python
@torch.no_grad()
def measure_pruning_rate(model, dataloader, beam_width=10):
    """
    Measure what fraction of positive items would be pruned by beam search.
    An item is 'pruned' if ANY of its tokens falls outside top-B at its step.
    """
    total_items = 0
    pruned_items = 0

    for batch in dataloader:
        input_ids, target_ids, target_mask = batch
        logits = model(input_ids).logits  # (batch, seq_len, vocab)
        probs = torch.softmax(logits, dim=-1)

        # Top-B threshold at each position
        topk_probs, _ = torch.topk(probs, beam_width, dim=-1)
        beta_B = topk_probs[:, :, -1]

        # Target token probabilities
        target_probs = probs.gather(-1, target_ids.unsqueeze(-1)).squeeze(-1)

        # A token is "pruned" if its prob < beta_B at that step
        token_pruned = (target_probs < beta_B) & target_mask.bool()

        # An item is pruned if ANY token in its sequence is pruned
        item_pruned = token_pruned.any(dim=-1)

        total_items += item_pruned.size(0)
        pruned_items += item_pruned.sum().item()

    rate = pruned_items / total_items
    print(f"Pruning rate: {rate:.2%} ({pruned_items}/{total_items} items)")
    return rate
```

Expected output before BEAR: `Pruning rate: 72.35% (5124/7083 items)`
Expected output after BEAR:  `Pruning rate: 48.12% (3409/7083 items)`

**Example 3: Integrating BEAR with HuggingFace Trainer**

User: "I'm using HuggingFace Trainer for my recommender fine-tuning. How do I plug BEAR in?"

Approach:
1. Subclass Trainer and override `compute_loss`
2. Add BEAR hyperparameters to training arguments
3. Keep everything else (data collator, evaluation, checkpointing) unchanged

```python
from transformers import Trainer

class BEARTrainer(Trainer):
    def __init__(self, *args, beam_width=10, lambda_reg=1.0, xi=1.0, **kwargs):
        super().__init__(*args, **kwargs)
        self.beam_width = beam_width
        self.lambda_reg = lambda_reg
        self.xi = xi

    def compute_loss(self, model, inputs, return_outputs=False, **kwargs):
        labels = inputs.pop("labels")
        outputs = model(**inputs)
        logits = outputs.logits

        # Shift logits and labels for next-token prediction
        shift_logits = logits[..., :-1, :].contiguous()
        shift_labels = labels[..., 1:].contiguous()

        loss = bear_loss(
            shift_logits, shift_labels,
            beam_width=self.beam_width,
            lambda_reg=self.lambda_reg,
            xi=self.xi
        )
        return (loss, outputs) if return_outputs else loss
```

## Best Practices

- **Do:** Match the beam width B in BEAR training to the beam width used at inference. Mismatched values weaken the guarantee.
- **Do:** Start with `lambda=1.0` and `xi=1.0`, then tune lambda in {0.1, 0.5, 1.0, 2.0, 5.0} on validation NDCG. The paper finds lambda=1.0 works well across datasets.
- **Do:** Use constrained beam search at inference, restricting generation to valid item tokens from your catalog. Unconstrained beam search will waste beam slots on invalid sequences.
- **Do:** Monitor both the SFT loss and the regularization loss separately during training to ensure neither dominates.
- **Avoid:** Setting xi too close to 0 — this makes the sigmoid nearly a step function, causing gradient issues. Values in [0.5, 2.0] work well.
- **Avoid:** Applying BEAR when using greedy decoding or sampling at inference. The regularization specifically targets beam search behavior and adds no value for other decoding strategies.

## Error Handling

- **Numerical instability in log probabilities:** Always add a small epsilon (1e-10) inside `torch.log()` calls to avoid `-inf` values when token probabilities are near zero.
- **Memory spikes from topk on large vocabularies:** `torch.topk` on (batch, seq_len, vocab_size) tensors can be expensive. If OOM occurs, compute the regularization term only on the item-token positions (using a mask) rather than the full sequence including prompt tokens.
- **Regularization loss exploding:** If `L_reg` grows much faster than `L_SFT`, reduce lambda or increase xi. A healthy ratio is `|L_reg| < 2 * |L_SFT|` during training.
- **No improvement in pruning rate:** Verify that your item token sequences are correctly aligned with the logit positions. Off-by-one errors in label shifting are common — the logits at position t predict the token at position t+1.
- **Constrained beam search not finding items:** Ensure your item vocabulary trie or prefix tree is correctly constructed. BEAR helps items survive beam search, but the constraint structure must contain them in the first place.

## Limitations

- BEAR assumes items are represented as discrete token sequences. It does not apply to embedding-based or collaborative filtering recommendation systems.
- The top-B necessary condition is relaxed, not sufficient — BEAR reduces but does not eliminate incorrect pruning. Some items with long token sequences may still be pruned if multiple tokens are near the boundary.
- Effectiveness scales with beam width B. Very small beam widths (B=1, i.e., greedy) leave no room for the regularizer to help; very large beam widths (B>50) rarely prune incorrectly anyway.
- BEAR adds a training-time constraint but does not modify the inference procedure. If your bottleneck is inference latency rather than retrieval quality, BEAR alone won't help.
- The method is validated on Amazon product recommendation datasets. Generalization to conversational recommendation, news recommendation, or other domains with very long item identifiers may require tuning.

## Reference

**Paper:** [BEAR: Towards Beam-Search-Aware Optimization for Recommendation with Large Language Models](https://arxiv.org/abs/2601.22925v1) (Yang et al., 2026)
**Key insight:** Section 3.2 derives the pruning margin `Delta_t^B` and sigmoid surrogate (Equations 3.2-3.5). Section 4.3 shows pruning rate analysis demonstrating ~70% of retrieval failures come from the training-inference mismatch BEAR addresses.