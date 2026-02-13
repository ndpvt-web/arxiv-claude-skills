---
name: "fast-slow-training-multimodal-visual"
description: "Implement DualSpeed fast-slow training for multimodal LLMs with visual token pruning. Use when: 'speed up MLLM training', 'reduce visual tokens during training', 'efficient multimodal fine-tuning', 'implement DualSpeed training', 'prune visual tokens for training efficiency', 'train LLaVA faster with token pruning'"
---

# Fast-Slow Efficient Training for Multimodal LLMs via Visual Token Pruning (DualSpeed)

This skill enables Claude to help users implement the DualSpeed dual-mode training framework, which accelerates multimodal LLM training by 2-4x while retaining over 99% of original performance. The core idea: train primarily on aggressively pruned visual token sequences (fast-mode, ~90% tokens removed) for speed, while occasionally training on full sequences (slow-mode, ~10% of batches) with self-distillation to eliminate the training-inference mismatch that naive token pruning causes.

## When to Use

- When the user wants to fine-tune or train a multimodal LLM (LLaVA, InternVL, etc.) and needs to reduce training cost or time
- When the user asks how to prune visual tokens during training without degrading inference quality
- When implementing a dual-mode training loop that alternates between pruned and full visual sequences
- When the user wants to add a mode isolator (learnable soft prompt) to separate pruned-mode and full-mode behavior in a single model
- When applying self-distillation where a pruned-sequence teacher guides a full-sequence student within the same model
- When the user asks about reducing GPU memory or FLOPs for vision-language model training

## Key Technique

**The Problem:** Multimodal LLMs process hundreds of visual tokens per image (e.g., 576 for a 336x336 image with ViT-L/14 at patch size 14). Training on these full sequences is expensive. Visual Token Pruning (VTP) can remove ~90% of tokens at inference with minimal quality loss, but naively applying VTP during training creates a mismatch -- the model learns to expect pruned inputs but receives full sequences at inference, causing 3-4% performance degradation.

**The Solution (DualSpeed):** A dual-mode training framework with three key components. (1) **Fast-mode** is the primary training pathway, applying a pluggable VTP method (attention-based, diversity-based, or conditional-diversity-based) to prune ~90% of visual tokens, yielding large speedups. A **mode isolator** -- a learnable soft prompt of length 4, prepended to pruned sequences -- lets the model distinguish between pruned and full inputs. (2) **Slow-mode** activates on ~10% of batches, training on full unpruned sequences. It uses **self-distillation**: the fast-mode processes the pruned version (without gradients) as a teacher, and the slow-mode processes the full version as a student, aligning their output distributions via KL-divergence loss. (3) During inference, the model uses full sequences without the mode isolator, exactly matching the slow-mode training distribution.

**Why it works:** Roughly 90% of visual tokens are redundant for learning. The fast-mode exploits this to train cheaply. The slow-mode + distillation closes the training-inference gap at minimal cost (only 10% of batches). The mode isolator prevents the two modes from interfering with each other inside a single set of weights. The result: 2.1x speedup on LLaVA-1.5 and 4.0x on LLaVA-NeXT, retaining 99%+ performance across 9 benchmarks.

## Step-by-Step Workflow

1. **Set up the base multimodal model architecture.** Start with a standard MLLM pipeline: visual encoder (e.g., CLIP ViT-L/14) -> projection layer -> LLM decoder. Ensure the visual encoder outputs a fixed grid of patch tokens (e.g., 576 tokens for 24x24 patches).

2. **Implement the visual token pruning plugin.** Choose a VTP method -- diversity-based pruning (DivPrune) performs best. The pruner takes the full set of N visual tokens and selects the top K tokens (where K = N * (1 - pruning_ratio)). For 90% pruning of 576 tokens, keep 58 tokens. Implement as a module that can be inserted between the visual encoder and the projection layer.

   ```python
   class DiversityPruner(nn.Module):
       def __init__(self, pruning_ratio=0.9):
           super().__init__()
           self.keep_ratio = 1.0 - pruning_ratio

       def forward(self, visual_tokens):
           # visual_tokens: (B, N, D)
           k = max(1, int(visual_tokens.size(1) * self.keep_ratio))
           # Iterative farthest-point sampling for diversity
           selected = self._diversity_select(visual_tokens, k)
           return selected

       def _diversity_select(self, tokens, k):
           B, N, D = tokens.shape
           # Initialize with token closest to mean
           centroids = tokens.mean(dim=1, keepdim=True)  # (B, 1, D)
           dists = torch.cdist(tokens, centroids).squeeze(-1)  # (B, N)
           indices = [dists.argmax(dim=1)]
           for _ in range(k - 1):
               selected_pts = tokens[torch.arange(B), indices[-1]].unsqueeze(1)
               new_dists = torch.cdist(tokens, selected_pts).squeeze(-1)
               dists = torch.min(dists, new_dists)
               indices.append(dists.argmax(dim=1))
           idx = torch.stack(indices, dim=1)  # (B, k)
           return tokens.gather(1, idx.unsqueeze(-1).expand(-1, -1, D))
   ```

3. **Add the mode isolator (learnable soft prompt).** Create a learnable parameter of shape `(l, D)` where `l=4` and `D` is the hidden dimension. In fast-mode, prepend this to the pruned visual tokens before feeding to the LLM.

   ```python
   class ModeIsolator(nn.Module):
       def __init__(self, hidden_dim, prompt_length=4):
           super().__init__()
           self.prompt = nn.Parameter(torch.randn(1, prompt_length, hidden_dim) * 0.02)

       def forward(self, pruned_visual_embeds):
           # pruned_visual_embeds: (B, K, D)
           B = pruned_visual_embeds.size(0)
           prefix = self.prompt.expand(B, -1, -1)
           return torch.cat([prefix, pruned_visual_embeds], dim=1)
   ```

4. **Implement the dual-mode forward pass.** On each batch, sample a Bernoulli random variable with probability `r=0.1`. If 0 (90% of the time), run fast-mode: prune tokens, prepend mode isolator, compute standard cross-entropy loss. If 1 (10% of the time), run slow-mode: compute full-sequence student loss plus distillation loss from the fast-mode teacher.

   ```python
   def dual_forward(model, pruner, isolator, images, text_ids, labels, r=0.1, tau=2.0, alpha=0.5):
       visual_tokens = model.encode_vision(images)  # (B, N, D)
       pruned_tokens = pruner(visual_tokens)
       pruned_embeds = isolator(model.project(pruned_tokens))

       is_slow = torch.rand(1).item() < r

       if not is_slow:
           # Fast-mode: train on pruned sequence
           logits = model.lm_forward(pruned_embeds, text_ids)
           loss = F.cross_entropy(logits.view(-1, logits.size(-1)), labels.view(-1))
       else:
           # Slow-mode: train on full sequence with distillation
           full_embeds = model.project(visual_tokens)
           student_logits = model.lm_forward(full_embeds, text_ids)
           ce_loss = F.cross_entropy(student_logits.view(-1, student_logits.size(-1)), labels.view(-1))

           # Teacher: fast-mode (no grad)
           with torch.no_grad():
               teacher_logits = model.lm_forward(pruned_embeds, text_ids)

           # KL-divergence distillation
           teacher_probs = F.softmax(teacher_logits / tau, dim=-1)
           student_log_probs = F.log_softmax(student_logits / tau, dim=-1)
           kl_loss = F.kl_div(student_log_probs, teacher_probs, reduction='batchmean') * (tau ** 2)

           loss = (1 - alpha) * ce_loss + alpha * kl_loss

       return loss
   ```

5. **Configure the training schedule.** During pretraining (projector-only, LLM frozen), use only fast-mode for maximum speedup (up to 5.8x). During supervised fine-tuning (SFT), activate dual-mode with `r=0.1`. The mode isolator and pruner parameters are trained alongside the model.

6. **Set the pruning ratio.** Use 90% as the default. Performance is stable across 50-90% but drops sharply above 90%. For higher-resolution models (LLaVA-NeXT with 2880 tokens), 90% pruning yields even larger speedups (4x) because the absolute token savings are greater.

7. **Configure inference.** At inference time, disable the pruner and mode isolator entirely. Feed full unpruned visual sequences directly to the model. The slow-mode training ensures the model handles full sequences correctly despite being trained primarily on pruned ones.

8. **Validate with targeted benchmarks.** Evaluate on VQAv2, GQA, ScienceQA, TextVQA, POPE, MME, MMBench, and SEED-Bench. Compare against the full-token baseline. Performance retention should be 99%+ across benchmarks. If any benchmark drops more than 2%, reduce the pruning ratio or increase the slow-mode probability `r`.

## Concrete Examples

**Example 1: Adding DualSpeed to an existing LLaVA training script**

User: "I have a LLaVA-1.5-7B training script that takes too long. Can you add DualSpeed to speed it up?"

Approach:
1. Identify where visual tokens are produced (after the CLIP encoder + projection).
2. Insert a `DiversityPruner(pruning_ratio=0.9)` between the visual encoder output and the LLM input.
3. Add a `ModeIsolator(hidden_dim=4096, prompt_length=4)` that prepends a learnable prefix in fast-mode.
4. Modify the training loop to sample fast/slow mode per batch with `r=0.1`.
5. In slow-mode batches, add KL-divergence distillation loss between the fast-mode teacher and full-sequence student.
6. Remove the pruner and isolator at inference time.

Output (key training loop diff):
```python
# Before: single-mode training
visual_embeds = model.project(model.encode_vision(images))
logits = model.lm_forward(visual_embeds, text_ids)
loss = ce_loss(logits, labels)

# After: DualSpeed dual-mode training
visual_tokens = model.encode_vision(images)
pruned = pruner(visual_tokens)
pruned_embeds = isolator(model.project(pruned))

if random.random() > 0.1:  # fast-mode (90% of batches)
    logits = model.lm_forward(pruned_embeds, text_ids)
    loss = ce_loss(logits, labels)
else:  # slow-mode (10% of batches)
    full_embeds = model.project(visual_tokens)
    student_logits = model.lm_forward(full_embeds, text_ids)
    with torch.no_grad():
        teacher_logits = model.lm_forward(pruned_embeds, text_ids)
    loss = (0.5 * ce_loss(student_logits, labels)
          + 0.5 * kl_div_loss(student_logits, teacher_logits, tau=2.0))
```

**Example 2: Choosing a pruning strategy for a high-resolution model**

User: "I'm training InternVL with 2880 visual tokens per image. Which VTP method should I use and what ratio?"

Approach:
1. High-resolution models benefit most from DualSpeed because absolute token savings are larger.
2. Recommend diversity-based pruning (DivPrune) -- it outperforms attention-based (FasterVLM) by ~1.5% in the paper's experiments.
3. Start with 90% pruning (2880 -> 288 tokens). This gives ~4x training speedup.
4. If the task is text-heavy (e.g., OCR, document understanding), lower to 80% (576 tokens) to preserve fine-grained spatial information.

Output:
```
Recommended config:
  - VTP method: DiversityPruner (farthest-point sampling)
  - Pruning ratio: 90% (2880 -> 288 tokens)
  - Slow-mode ratio: r = 0.1
  - Mode isolator length: l = 4
  - Expected speedup: ~4x
  - Expected performance retention: 99%+

For text-heavy tasks (TextVQA, DocVQA), reduce pruning to 80%.
```

**Example 3: Debugging a performance drop after adding DualSpeed**

User: "I added visual token pruning to training and my MMBench score dropped by 5%. What's wrong?"

Approach:
1. Check if slow-mode is enabled. Training only with fast-mode causes a 3-4% drop due to training-inference mismatch.
2. Verify the mode isolator is active. Without it, the model conflates pruned and full-sequence modes, losing ~0.5%.
3. Confirm the distillation loss is computed correctly. The teacher should process pruned tokens without gradients; the student processes full tokens.
4. Check the pruning ratio. If above 90%, reduce to 90%.
5. Verify that at inference time the pruner and mode isolator are fully disabled.

Output:
```
Diagnostic checklist:
  [?] Is slow-mode active (r > 0)?         -> If not, set r=0.1
  [?] Is mode isolator prepended in fast-mode only? -> Check forward pass
  [?] Is teacher using no_grad()?           -> Verify detach/no_grad
  [?] Is pruning ratio <= 90%?              -> Reduce if higher
  [?] Is inference using full tokens?       -> Disable pruner at eval
```

## Best Practices

- **Do:** Use diversity-based pruning (DivPrune) as the default VTP plugin. It consistently outperforms attention-based methods in the training context.
- **Do:** Keep the slow-mode probability at 10% (`r=0.1`). This balances training-inference consistency against speed gains. Higher values waste compute; lower values degrade inference quality.
- **Do:** Set the mode isolator prompt length to 4. Ablations show diminishing returns beyond this length with no measurable benefit at `l=8` or `l=16`.
- **Do:** Disable all pruning components at inference time. The model's slow-mode training ensures it handles full sequences natively.
- **Avoid:** Pruning above 90% of visual tokens. Performance drops sharply beyond this threshold (>3% degradation).
- **Avoid:** Applying DualSpeed during projector-only pretraining with slow-mode enabled. In the pretraining phase (LLM frozen), use fast-mode exclusively for maximum speedup since the LLM isn't updating weights.
- **Avoid:** Using a distillation temperature below 1.0. The paper uses `tau=2.0` to soften distributions for effective knowledge transfer.

## Error Handling

| Issue | Cause | Fix |
|-------|-------|-----|
| Large gap between fast-mode train loss and slow-mode train loss | Modes are learning different distributions | Increase `r` from 0.1 to 0.15; verify mode isolator is separating the two pathways |
| Inference performance much worse than fast-mode eval | Training-inference mismatch; slow-mode undertrained | Increase `r` to 0.2 or train for more steps; verify distillation loss is active |
| OOM on slow-mode batches | Full-sequence batches + teacher forward use more memory | Use gradient checkpointing on slow-mode batches; reduce batch size for slow-mode only |
| Mode isolator parameters not updating | Optimizer not including isolator params | Ensure `isolator.parameters()` is in the optimizer param group |
| Pruner selects same tokens every batch | Deterministic pruning without diversity sampling | Verify farthest-point sampling implementation; add minor noise to distances |

## Limitations

- **Task-specific sensitivity:** Text-heavy tasks (OCR, document QA) are more sensitive to visual token pruning because fine-grained spatial tokens carry critical information. For these tasks, reduce pruning ratio to 70-80%.
- **Architecture dependency:** The technique assumes a standard vision-encoder + projector + LLM pipeline. Architectures with interleaved visual attention (e.g., Flamingo-style cross-attention) would need a different pruning insertion point.
- **Pruner overhead:** The VTP module itself adds some compute. For very small models or very short sequences, the pruning overhead may negate the speedup from reduced tokens.
- **Single-image focus:** The paper evaluates on single-image tasks. Multi-image or video scenarios with thousands of tokens may need different pruning ratios and slow-mode frequencies.
- **No dynamic pruning rate:** The pruning ratio is fixed per run. Adaptive pruning that adjusts per-sample based on image complexity is not addressed.

## Reference

**Paper:** [Fast-Slow Efficient Training for Multimodal Large Language Models via Visual Token Pruning](https://arxiv.org/abs/2602.03815v1) (Zhang et al., 2026). Look for: Table 2 (pruning ratio ablation), Table 3 (component ablation showing mode isolator + distillation contributions), and Section 3.2 (the dual-mode training algorithm with loss formulations).