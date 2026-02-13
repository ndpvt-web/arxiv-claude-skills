---
name: "sed-sft-selectively-encouraging-diversity"
description: "Implement SED-SFT selective entropy regularization to combat mode collapse in LLM supervised fine-tuning. Use when: 'add diversity regularization to SFT', 'fix mode collapse in fine-tuning', 'implement SED-SFT loss', 'selective entropy masking for training', 'improve SFT for downstream RL', 'encourage diverse generation in fine-tuning'"
---

# SED-SFT: Selectively Encouraging Diversity in Supervised Fine-Tuning

This skill enables Claude to implement the SED-SFT loss function -- a selective entropy regularization technique that combats mode collapse during supervised fine-tuning of large language models. Instead of uniformly applying entropy penalties across all tokens (which degrades accuracy), SED-SFT identifies tokens where the model is already overconfident and selectively injects diversity pressure only there. This preserves accuracy on well-learned tokens while expanding the exploration space for downstream reinforcement learning.

## When to Use

- When the user is building an SFT training pipeline and wants to prevent mode collapse before RL stages (RLHF, GRPO, PPO)
- When the user asks how to add entropy regularization to cross-entropy loss without hurting accuracy
- When the user reports their fine-tuned model produces repetitive, low-diversity outputs
- When the user wants to implement a custom loss function that balances accuracy and generation diversity
- When the user is preparing an SFT checkpoint specifically optimized for subsequent RL training
- When the user asks to reproduce or adapt the SED-SFT paper's training setup
- When the user wants to compare CE, GEM, and SED loss functions for SFT

## Key Technique

**The Mode Collapse Problem.** Standard SFT with cross-entropy (CE) loss trains the model to maximize the probability of a single reference token at each position. Over training, this collapses the output distribution to a sharp peak around the reference -- the model becomes overconfident and loses the distributional diversity needed for RL exploration. Naive entropy maximization (adding `-H(p)` uniformly) partially fixes this but hurts accuracy by penalizing tokens the model has correctly learned with high confidence.

**Selective Entropy Regularization.** SED-SFT introduces a confidence-aware binary mask that identifies which tokens need diversity encouragement. For each token position, the model computes softmax probabilities, extracts the top-k values, and sums them. If this cumulative top-k probability falls below a threshold (default 0.95), the token is considered "already exploring" and left alone. If the top-k cumsum exceeds the threshold, the model is overconfident there, and an entropy penalty is applied. The penalty term uses the model's predicted probability for the ground-truth token: `target_prob * (target_prob - 1) + 0.25`, scaled by `entropy_penalty_scale`. This creates a gentle quadratic push away from probability extremes.

**Why It Works.** By masking out tokens that already have diverse predictions, SED-SFT avoids the accuracy-diversity tradeoff. High-confidence correct predictions are left undisturbed. Only tokens where the model has "collapsed" onto a narrow distribution receive the regularization signal. The result: diversity improves where it matters (enabling RL exploration), accuracy is preserved where it matters (correct reasoning steps), and computational overhead is negligible since the mask is a simple top-k cumsum operation.

## Step-by-Step Workflow

1. **Identify the training loop and loss computation.** Locate where the CE loss is computed in the user's SFT training code -- typically in a custom `Trainer` subclass or the `compute_loss` method. Confirm the code has access to raw logits (not just the scalar loss).

2. **Compute softmax probabilities from logits.** After the forward pass produces logits of shape `(batch, seq_len, vocab_size)`, apply `F.softmax(logits, dim=-1)` to get token-level probability distributions. Shift logits and labels by one position to align predictions with targets (standard autoregressive setup).

3. **Build the selective mask via top-k cumulative probability.** For each token position, extract the top-k probabilities (default k=10), sum them, and compare against `cumsum_threshold` (default 0.95). Positions where `topk_sum >= cumsum_threshold` are flagged as overconfident -- these receive the entropy penalty. Positions below the threshold are already diverse and are masked out.

4. **Extract target token probabilities.** Using the ground-truth labels, gather the model's predicted probability for the correct token at each position: `target_probs = probs.gather(-1, labels.unsqueeze(-1)).squeeze(-1)`. Exclude padding tokens (label == -100).

5. **Compute the selective entropy penalty.** For masked (overconfident) positions, compute: `penalty = target_prob * (target_prob - 1) + 0.25`. This is a concave quadratic that equals 0.25 when `target_prob = 0.5` and approaches 0 at the extremes, creating a gentle regularization force. Scale by `entropy_penalty_scale` (default 0.2).

6. **Combine with the base CE loss.** Add the penalty to the per-token CE loss: `total_loss = ce_loss + entropy_penalty_scale * penalty * mask`. Apply the standard padding mask and reduce (mean over non-padding tokens).

7. **Configure hyperparameters.** Set `top_k=10`, `cumsum_threshold=0.95` (or the paper's alternative value of 0.7 for more aggressive diversity), `entropy_penalty_scale=0.2`, and `beta=0.7`. These are the defaults from the reference implementation.

8. **Integrate with the training framework.** If using HuggingFace Transformers, subclass `Trainer` and override `compute_loss`. If using a custom loop, replace the loss computation directly. Ensure gradient checkpointing and mixed precision (bf16) are enabled for efficiency.

9. **Monitor diversity metrics during training.** Log the mask ratio (fraction of tokens receiving the penalty) and per-step entropy alongside the training loss. A healthy mask ratio is typically 30-70% -- if it drops near 0%, the threshold is too aggressive; if near 100%, it is too permissive.

10. **Validate before RL.** After SFT, sample multiple completions for the same prompts and measure pass@k, distinct-n, or self-BLEU to confirm diversity has improved relative to a CE-only baseline. Then proceed to RL training (GRPO, PPO, etc.) on the SED-SFT checkpoint.

## Concrete Examples

**Example 1: Adding SED-SFT Loss to a HuggingFace Trainer**

User: "I'm fine-tuning Llama-3 for math reasoning with SFT before GRPO. How do I add the SED-SFT loss to prevent mode collapse?"

Approach:
1. Subclass `transformers.Trainer` to override `compute_loss`
2. Implement the selective mask and entropy penalty
3. Return the combined loss

Output:
```python
import torch
import torch.nn.functional as F
from transformers import Trainer

class SEDSFTTrainer(Trainer):
    def __init__(self, *args, top_k=10, cumsum_threshold=0.95,
                 entropy_penalty_scale=0.2, **kwargs):
        super().__init__(*args, **kwargs)
        self.top_k = top_k
        self.cumsum_threshold = cumsum_threshold
        self.entropy_penalty_scale = entropy_penalty_scale

    def compute_loss(self, model, inputs, return_outputs=False, **kwargs):
        labels = inputs.pop("labels")
        outputs = model(**inputs)
        logits = outputs.logits

        # Shift for autoregressive alignment
        shift_logits = logits[..., :-1, :].contiguous()
        shift_labels = labels[..., 1:].contiguous()

        # Standard CE loss (per-token)
        ce_loss = F.cross_entropy(
            shift_logits.view(-1, shift_logits.size(-1)),
            shift_labels.view(-1),
            ignore_index=-100,
            reduction='none'
        ).view(shift_labels.shape)

        # Compute probabilities and selective mask
        probs = F.softmax(shift_logits, dim=-1)
        topk_probs, _ = probs.topk(self.top_k, dim=-1)
        topk_cumsum = topk_probs.sum(dim=-1)  # (batch, seq_len)

        # Mask: True where model is overconfident (needs diversity)
        overconfident_mask = (topk_cumsum >= self.cumsum_threshold)

        # Target token probabilities
        valid_labels = shift_labels.clone()
        valid_labels[valid_labels == -100] = 0
        target_probs = probs.gather(-1, valid_labels.unsqueeze(-1)).squeeze(-1)

        # Selective entropy penalty (quadratic form)
        penalty = target_probs * (target_probs - 1.0) + 0.25

        # Apply mask: only penalize overconfident tokens
        padding_mask = (shift_labels != -100).float()
        selective_penalty = penalty * overconfident_mask.float() * padding_mask

        # Combined loss
        total_loss = (ce_loss * padding_mask + self.entropy_penalty_scale * selective_penalty)
        total_loss = total_loss.sum() / padding_mask.sum().clamp(min=1)

        return (total_loss, outputs) if return_outputs else total_loss
```

**Example 2: Implementing SED-SFT in a Custom PyTorch Training Loop**

User: "I have a custom training loop for SFT. Show me how to add the SED selective masking."

Approach:
1. Write a standalone `sed_sft_loss` function
2. Drop it into the existing training loop replacing `F.cross_entropy`

Output:
```python
def sed_sft_loss(logits, labels, top_k=10, cumsum_threshold=0.95,
                 entropy_penalty_scale=0.2):
    """
    SED-SFT loss: CE + selective entropy regularization.

    Args:
        logits: (batch, seq_len, vocab_size) raw model output
        labels: (batch, seq_len) target token IDs, -100 for padding
        top_k: number of top probabilities to sum for confidence check
        cumsum_threshold: tokens with top-k cumsum above this get penalized
        entropy_penalty_scale: weight of the entropy penalty term
    """
    # Shift for next-token prediction
    logits = logits[:, :-1, :].contiguous()
    labels = labels[:, 1:].contiguous()

    batch_size, seq_len, vocab_size = logits.shape
    padding_mask = (labels != -100).float()

    # Per-token CE loss
    ce_loss = F.cross_entropy(
        logits.view(-1, vocab_size), labels.view(-1),
        ignore_index=-100, reduction='none'
    ).view(batch_size, seq_len)

    # Selective mask construction
    probs = F.softmax(logits, dim=-1)
    topk_vals, _ = probs.topk(top_k, dim=-1)
    confidence = topk_vals.sum(dim=-1)
    overconfident = (confidence >= cumsum_threshold).float()

    # Entropy penalty on overconfident tokens
    safe_labels = labels.clamp(min=0)
    target_p = probs.gather(-1, safe_labels.unsqueeze(-1)).squeeze(-1)
    penalty = target_p * (target_p - 1.0) + 0.25

    # Combine
    loss = ce_loss + entropy_penalty_scale * penalty * overconfident
    loss = (loss * padding_mask).sum() / padding_mask.sum().clamp(min=1)
    return loss

# In training loop, replace:
#   loss = F.cross_entropy(logits, labels)
# with:
#   loss = sed_sft_loss(logits, labels)
```

**Example 3: Tuning the Cumsum Threshold for Aggressive vs. Conservative Diversity**

User: "My SFT model still produces repetitive outputs after using SED-SFT. How do I increase diversity?"

Approach:
1. Lower `cumsum_threshold` from 0.95 to 0.7 (the paper's alternative setting)
2. Increase `entropy_penalty_scale` from 0.2 to 0.3-0.5
3. Monitor mask ratio to verify more tokens are being regularized

Output:
```python
# Conservative (default) -- penalizes only the most overconfident tokens
trainer = SEDSFTTrainer(..., cumsum_threshold=0.95, entropy_penalty_scale=0.2)

# Aggressive -- penalizes a broader set of tokens, stronger push for diversity
trainer = SEDSFTTrainer(..., cumsum_threshold=0.7, entropy_penalty_scale=0.4)

# To monitor: log mask ratio each step
mask_ratio = (overconfident_mask * padding_mask).sum() / padding_mask.sum()
print(f"Mask ratio: {mask_ratio:.2%}")  # Target: 30-70%
```

If mask ratio is above 90%, the model is overconfident almost everywhere -- lower `cumsum_threshold` further or check if the base model needs more pretraining data. If mask ratio is below 10%, raise `cumsum_threshold` closer to 1.0.

## Best Practices

- **Do:** Start with the default hyperparameters (`top_k=10`, `cumsum_threshold=0.95`, `entropy_penalty_scale=0.2`) and only adjust if diversity metrics are unsatisfactory after evaluation.
- **Do:** Log the mask ratio during training as a diagnostic. It directly tells you what fraction of tokens are receiving diversity pressure.
- **Do:** Evaluate diversity with sampling-based metrics (pass@k with k=10+, distinct-n, or self-BLEU) rather than greedy decoding accuracy alone.
- **Do:** Use bf16 mixed precision -- the softmax and top-k operations are numerically stable in bf16 and the overhead is minimal.
- **Avoid:** Applying SED-SFT to padding tokens or special tokens. Always mask them out (label == -100) before computing the penalty.
- **Avoid:** Setting `entropy_penalty_scale` above 1.0. The penalty should be a gentle nudge, not a dominant signal -- values above 0.5 risk destabilizing training.
- **Avoid:** Using SED-SFT without a subsequent RL stage. The diversity gains are specifically designed to improve RL exploration. If the final model is the SFT checkpoint itself, standard CE may yield better greedy accuracy.

## Error Handling

- **OOM on top-k computation:** The `probs.topk(k, dim=-1)` call over the full vocabulary can be memory-intensive. If OOM occurs, reduce `top_k` from 10 to 5, or compute the mask on chunked sequences. The paper uses k=10 but smaller values still capture the confidence signal.
- **NaN in penalty term:** If `target_probs` contains zeros (from padding or numerical underflow), the penalty formula `p*(p-1)+0.25` is still numerically stable (it equals 0.25 at p=0). However, ensure the padding mask is applied correctly before the loss reduction to avoid dividing by zero.
- **Mask ratio stuck at 0% or 100%:** This indicates the threshold is misaligned with the model's confidence distribution. Print a histogram of `topk_cumsum` values at initialization and set the threshold at the 50th-70th percentile.
- **No diversity improvement after training:** Verify the penalty is actually reaching the optimizer -- check that `selective_penalty.requires_grad` is True and that the penalty term is non-zero in the total loss.

## Limitations

- SED-SFT is designed for the SFT-then-RL pipeline. If no RL stage follows, the diversity gains may not translate to better final performance and could slightly reduce greedy accuracy.
- The paper validates exclusively on mathematical reasoning benchmarks (MATH, GSM8K, etc.). Effectiveness on creative writing, code generation, or instruction following has not been established.
- The top-k cumsum mask is a heuristic proxy for "overconfidence." It works well empirically but may not optimally identify which tokens need diversity in all domains.
- The quadratic penalty `p*(p-1)+0.25` is one possible regularization shape. Other functional forms (e.g., direct entropy `-p*log(p)`) might work better in different settings but were not explored in the paper.
- Computational overhead is low but non-zero: the top-k operation over the vocabulary adds ~5-10% to the loss computation time. For very large vocabularies (100k+), consider the Triton-optimized kernel from the reference implementation.

## Reference

**Paper:** [SED-SFT: Selectively Encouraging Diversity in Supervised Fine-Tuning](https://arxiv.org/abs/2602.07464v1) (Chen, Liu, Meng, 2026). Look for Section 3 (method) for the loss formulation and selective masking derivation, and Section 4 (experiments) for ablations on threshold and penalty scale values.

**Code:** [https://github.com/pppa2019/SED-SFT](https://github.com/pppa2019/SED-SFT) -- contains Triton-optimized kernels (`utils/sed_triton_loss.py`) and training scripts with DeepSpeed integration.