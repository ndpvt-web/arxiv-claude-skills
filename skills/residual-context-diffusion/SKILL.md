---
name: "residual-context-diffusion"
description: "Implement and apply Residual Context Diffusion (RCD) for diffusion language models. Converts wasted computation from remasked tokens into contextual residuals that improve accuracy 5-10 points with minimal overhead. Use when: 'implement RCD for a diffusion LLM', 'add residual context to a remasking decoder', 'optimize dLLM denoising steps', 'recycle discarded token probabilities', 'reduce denoising iterations in block-wise decoding', 'train an RCD module on top of LLaDA or SDAR'."
---

# Residual Context Diffusion for Diffusion Language Models

This skill enables Claude to implement the Residual Context Diffusion (RCD) technique from Hu et al. (2026), which recycles wasted computation in diffusion Large Language Models (dLLMs). In standard block-wise dLLMs, a remasking mechanism keeps only the highest-confidence token predictions at each denoising step and discards the rest -- throwing away useful contextual signals. RCD captures those discarded probability distributions, converts them into weighted embedding residuals, and injects them back into the next denoising step via entropy-gated interpolation. This consistently yields 5-10 point accuracy gains, nearly doubles performance on hard reasoning tasks (AIME), and achieves equivalent accuracy in 4-5x fewer denoising steps.

## When to Use

- When implementing a diffusion language model training or inference pipeline that uses remasking (e.g., LLaDA, SDAR, MDLM block-wise decoders)
- When the user wants to improve accuracy of an existing dLLM without retraining the full model from scratch
- When building a denoising loop for masked token prediction and wants to carry forward information from discarded predictions
- When optimizing inference throughput by reducing the number of denoising steps needed to reach target accuracy
- When implementing the two-stage decoupled training pipeline to add RCD to a pre-trained dLLM checkpoint
- When the user asks about recycling intermediate logits/probabilities in iterative decoding systems

## Key Technique

**The problem:** Block-wise dLLMs decode by predicting all masked positions, committing only the top-m most confident tokens, and remasking the rest. The remasked positions lose all intermediate probability information -- their embeddings reset to the generic `[MASK]` token. This wastes the computation spent on those positions and discards contextual signals that could guide subsequent steps.

**The RCD solution:** Instead of resetting remasked positions to a bare `[MASK]` embedding, RCD computes a *contextual residual* for each position: a weighted sum over the vocabulary embedding table using that position's predicted probability distribution. This residual is then blended with the `[MASK]` embedding using the position's normalized Shannon entropy as a mixing weight. High-entropy (uncertain) positions get more residual injection because they carry diffuse but informative context; low-entropy (near-decided) positions get less because they're already close to commitment. The formula is: `e_i = (1 - alpha_i) * E([M]) + alpha_i * Delta_i`, where `Delta_i = sum_j p_ij * E_j` and `alpha_i = -sum_j p_ij * log(p_ij) / log(V)`.

**Training without memory blowup:** Naive backpropagation through the residual chain would require storing activations across all denoising steps -- infeasible for large models. RCD uses a decoupled two-stage approach: (1) fine-tune a lightweight reference model on your task data, freeze it; (2) train the target model to consume residual-enriched embeddings produced by the frozen reference model. This breaks the gradient chain while preserving the benefit. The target model can be converted with only ~1 billion training tokens.

## Step-by-Step Workflow

### Implementing RCD Training

1. **Prepare the reference model.** Initialize a copy of your pre-trained dLLM (can be a smaller variant, e.g., 1.7B for an 8B target). Fine-tune it on your downstream dataset using the standard masked language modeling objective. Freeze all parameters after convergence.

2. **Implement the residual vector computation.** For each masked position `i`, compute `Delta_i = sum_j p_ij * E_j` where `p` is the frozen reference model's softmax output and `E` is the target model's embedding codebook. This produces a dense vector in the same space as token embeddings.

   ```python
   # p_ref: (batch, seq_len, vocab_size) from frozen reference model
   # embed_weight: (vocab_size, hidden_dim) from target model
   delta = torch.matmul(p_ref, embed_weight)  # (batch, seq_len, hidden_dim)
   ```

3. **Compute entropy-based mixing weights.** Calculate normalized Shannon entropy for each position's probability distribution. This determines how much residual signal to inject.

   ```python
   log_p = torch.log(p_ref + 1e-10)
   entropy = -torch.sum(p_ref * log_p, dim=-1)  # (batch, seq_len)
   alpha = entropy / math.log(vocab_size)         # normalize to [0, 1]
   ```

4. **Construct residual-enriched embeddings.** For masked positions, interpolate between the `[MASK]` embedding and the residual vector. Unmasked positions use their standard token embeddings unchanged.

   ```python
   mask_embed = embed_weight[mask_token_id].unsqueeze(0).unsqueeze(0)
   alpha_expanded = alpha.unsqueeze(-1)  # (batch, seq_len, 1)
   enriched = (1 - alpha_expanded) * mask_embed + alpha_expanded * delta
   # Apply only to masked positions
   input_embeds = torch.where(is_masked.unsqueeze(-1), enriched, token_embeds)
   ```

5. **Forward pass through the target model.** Feed the residual-enriched embeddings (instead of raw token embeddings) into the target dLLM. Compute standard cross-entropy loss over the masked positions against ground-truth tokens.

6. **Train with standard hyperparameters.** Use AdamW optimizer (lr=1e-5, betas=(0.9, 0.999)), bfloat16 precision. Batch sizes: 96 for sequence length 8192 (long CoT) or 768 for sequence length 2048 (instruction following). Train for 5-10 epochs on ~1B tokens.

### Implementing RCD Inference

7. **Initialize the denoising loop with a warm start.** Before the first denoising step, run the reference model on the fully-masked block to obtain initial probability distributions and entropy weights. This gives the first iteration useful residual context instead of starting cold.

8. **At each denoising step, inject residuals before the forward pass.** Use the *previous step's* probability distributions (with temperature scaling) to construct residual vectors for currently-masked positions. Apply temperature `T_res` to logits before softmax to bridge the train-test distribution gap.

   ```python
   p_scaled = torch.softmax(logits_prev / T_res, dim=-1)
   delta = torch.matmul(p_scaled, embed_weight)
   alpha = normalized_entropy(p_scaled)
   # Inject into masked positions before model forward pass
   ```

9. **Select, commit, and remask as usual.** After the forward pass, select the top-m most confident predictions, commit those tokens, and remask the rest. The key difference from vanilla remasking: the remasked positions will receive residual context in the *next* step rather than being reset to bare `[MASK]`.

10. **Tune the residual temperature.** The `T_res` parameter controls entropy weight distribution during inference. Start with `T_res = 1.0` and sweep in [0.5, 2.0]. Lower temperatures sharpen residuals (more confident injection), higher temperatures smooth them (more uniform context).

## Concrete Examples

**Example 1: Adding RCD to an existing LLaDA-style dLLM**

User: "I have a LLaDA-8B checkpoint that uses global bidirectional denoising with remasking. I want to add RCD to improve its math reasoning accuracy."

Approach:
1. Fine-tune a copy of LLaDA-8B on your math dataset (standard MLM loss) for 5 epochs with lr=1e-5, batch_size=768, seq_len=2048. Freeze it as the reference model.
2. Modify the target model's embedding layer to accept residual-enriched inputs. Add a `ResidualInjector` module that takes reference model logits and produces enriched embeddings.
3. Train the target model for 5 epochs on the same data (~1B tokens). The reference model runs in inference mode (no grad) to produce probability distributions fed into the injector.
4. At inference, run one warm-start reference pass, then loop the denoising with residual injection at each step.

Output: LLaDA-8B-RCD that scores ~78% on GSM8K (up from ~76%) and ~37% on MinervaMath (up from ~31%).

**Example 2: Reducing denoising steps for faster SDAR inference**

User: "My SDAR-8B model with block size 64 takes too many denoising steps. I want to match its current accuracy in fewer steps."

Approach:
1. Use an SDAR-1.7B as the reference model (smaller model works -- the paper demonstrates cross-scale transfer).
2. Train the RCD target on ~1B tokens with sequence length 8192 for long CoT.
3. During inference, set `T_res = 1.0` and reduce denoising steps from the default. With RCD, you achieve the same accuracy at 4-5x fewer steps on AIME-level tasks.
4. Profile throughput: RCD adds negligible latency per step (one extra matmul for residual computation + entropy calculation), so the step reduction translates directly to wall-clock speedup.

Output: Equivalent MATH500 accuracy (~64%) reached in ~25% of the original denoising steps. On AIME24, accuracy jumps from ~7% to ~15% at the same step budget.

**Example 3: Implementing the ResidualInjector module from scratch**

User: "Write me the core RCD module I can drop into my dLLM codebase."

```python
import torch
import torch.nn as nn
import math

class ResidualContextInjector(nn.Module):
    """Injects contextual residuals from previous denoising step into masked embeddings."""

    def __init__(self, vocab_size: int, mask_token_id: int):
        super().__init__()
        self.vocab_size = vocab_size
        self.mask_token_id = mask_token_id
        self.log_vocab = math.log(vocab_size)

    def compute_residuals(
        self,
        prev_logits: torch.Tensor,      # (B, S, V) from previous step
        embed_weight: torch.Tensor,      # (V, D) target model embeddings
        T_res: float = 1.0,
    ) -> tuple[torch.Tensor, torch.Tensor]:
        """Compute residual vectors and entropy weights from previous-step logits."""
        p = torch.softmax(prev_logits / T_res, dim=-1)

        # Residual: probability-weighted embedding sum
        delta = torch.matmul(p, embed_weight)  # (B, S, D)

        # Normalized Shannon entropy as mixing weight
        log_p = torch.log(p + 1e-10)
        entropy = -torch.sum(p * log_p, dim=-1)  # (B, S)
        alpha = entropy / self.log_vocab           # (B, S) in [0, 1]

        return delta, alpha

    def inject(
        self,
        token_ids: torch.LongTensor,     # (B, S) current token ids (with [M])
        token_embeds: torch.Tensor,       # (B, S, D) standard embeddings
        delta: torch.Tensor,              # (B, S, D) residual vectors
        alpha: torch.Tensor,              # (B, S) entropy weights
    ) -> torch.Tensor:
        """Replace masked position embeddings with residual-enriched versions."""
        is_masked = (token_ids == self.mask_token_id)  # (B, S)
        mask_embed = token_embeds[is_masked]            # uses the [M] embedding

        alpha_m = alpha[is_masked].unsqueeze(-1)        # (N_masked, 1)
        delta_m = delta[is_masked]                      # (N_masked, D)

        enriched = (1 - alpha_m) * mask_embed + alpha_m * delta_m
        output = token_embeds.clone()
        output[is_masked] = enriched
        return output
```

## Best Practices

- **Do:** Use a smaller reference model when the target model is large. The paper shows SDAR-1.7B effectively guides SDAR-4B and SDAR-8B, saving significant memory.
- **Do:** Apply temperature scaling (`T_res`) during inference to bridge the distribution gap between training (where ground-truth masking ratios are sampled) and inference (where masking ratios decrease monotonically).
- **Do:** Run the warm-start reference pass before the first denoising step. Skipping it means the first iteration has no residual context, negating the benefit.
- **Do:** Keep the reference model frozen during Stage 2 training. Unfreezing it re-introduces the memory bottleneck the two-stage design was built to avoid.
- **Avoid:** Injecting residuals into unmasked (already-committed) positions. These should always use their committed token's embedding directly.
- **Avoid:** Using very low `T_res` values (< 0.3) which collapse the entropy weights and make residuals too sharp, losing the soft contextual signal that makes RCD effective.

## Error Handling

- **Reference model diverges from target model's embedding space:** If you use separately pre-trained models, the residual vectors (computed with the target's embeddings but the reference's probabilities) may be misaligned. Always ensure both models share the same tokenizer and embedding initialization, or derive the reference from the same checkpoint.
- **Memory overflow during Stage 2:** The frozen reference model must fit in memory alongside the target model. Use the smaller-model-as-reference strategy (1.7B ref for 8B target) or offload the reference model to CPU with selective GPU transfer per batch.
- **Entropy weights collapse to near-zero:** This happens when the reference model is overconfident (low entropy everywhere). Check that the reference was trained with standard hyperparameters and not overtrained. If needed, increase `T_res` to redistribute entropy mass.
- **No improvement on short-sequence tasks:** RCD benefits scale with the number of denoising steps. For tasks with very few masked positions or single-step decoding, the residual injection has minimal effect. Verify your task actually uses multi-step remasking.

## Limitations

- RCD requires a pre-trained dLLM that uses remasking-based iterative decoding. It does not apply to autoregressive models (GPT-style) or single-pass masked language models (BERT-style).
- The two-stage training requires maintaining two models in memory simultaneously (frozen reference + trainable target), roughly doubling GPU memory vs. standard fine-tuning unless you use a smaller reference.
- Gains are most pronounced on hard multi-step reasoning (AIME: +10pp) and decrease on simpler tasks (GSM8K: +2-6pp). For already-easy benchmarks, the overhead may not be justified.
- The technique assumes the reference model produces meaningful probability distributions. If the reference model performs poorly on your domain, the residuals will carry noise rather than signal.
- RCD does not change the model architecture -- it modifies only the input embedding construction. This means it cannot be applied at inference time without the corresponding RCD-aware training.

## Reference

**Paper:** Hu, Y., Singh, H., Maheswaran, M., Xi, H., & Hooper, C. (2026). *Residual Context Diffusion Language Models.* arXiv:2601.22954v1. [https://arxiv.org/abs/2601.22954v1](https://arxiv.org/abs/2601.22954v1)

**What to look for:** Section 3 for the full mathematical formulation of residual vectors and entropy-gated injection; Section 4 for the decoupled two-stage training algorithm (Algorithms 1 and 2); Section 5.3 for ablation studies on temperature scaling and warm-start effects; Tables 1-3 for comprehensive benchmark comparisons across SDAR and LLaDA variants.

**Code:** [https://github.com/yuezhouhu/residual-context-diffusion](https://github.com/yuezhouhu/residual-context-diffusion)