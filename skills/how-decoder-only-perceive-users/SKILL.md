---
name: "how-decoder-only-perceive-users"
description: "Implement Gradient-Guided Soft Masking (GGSM) attention strategies for adapting decoder-only LLMs to user representation learning. Covers attention mask design, causal-to-bidirectional training transitions, and contrastive user embedding pipelines. Use when: 'build a user embedding model from behavioral sequences', 'implement attention masking for user representations', 'train a decoder LLM for user profiling', 'add bidirectional attention to a causal model', 'implement contrastive learning for user behavior', 'design a gradient-guided mask scheduler'."
---

# Gradient-Guided Soft Masking for User Representation Learning

This skill enables Claude to implement **Gradient-Guided Soft Masking (GGSM)** — a technique for adapting decoder-only LLMs to produce high-quality user embeddings from heterogeneous behavioral sequences. Instead of naively switching a causal model to bidirectional attention (which destabilizes training), GGSM uses gradient magnitudes to compute a soft warmup mask that smoothly transitions attention visibility, followed by a linear scheduler that fully opens bidirectional attention. The result is stable training and superior user representations for downstream tasks like preference prediction, marketing sensitivity, and user cognition benchmarks.

## When to Use

- When the user wants to **encode user behavior sequences** (clicks, purchases, page views) into fixed-size embeddings using a decoder-only LLM
- When building a **contrastive learning pipeline** that trains user representations from behavioral logs
- When the user needs to **modify the attention mask** of a pretrained causal LLM to allow bidirectional or hybrid attention for embedding tasks
- When transitioning a **causal model to bidirectional attention** and encountering training instability (loss spikes, gradient explosions)
- When designing a **mask scheduling strategy** that gradually opens future-token attention during fine-tuning
- When implementing **user representation models** for recommendation, user profiling, or marketing sensitivity tasks on heterogeneous behavioral data

## Key Technique

### The Problem: Causal Masks Limit User Embeddings

Decoder-only LLMs use causal (left-to-right) attention masks during pretraining, meaning each token can only attend to previous tokens. When repurposing these models as user behavior encoders, this is suboptimal: the final representation cannot incorporate information from earlier tokens informed by later context. Bidirectional attention would let every position attend to every other, producing richer embeddings — but simply switching from causal to bidirectional attention breaks the pretrained model's learned attention patterns, causing training collapse.

### The Solution: Gradient-Guided Soft Masking + Linear Scheduling

GGSM addresses this in two phases. **Phase 1 (Gradient-Guided Pre-Warmup):** Before any mask transition, compute the gradient magnitude of the contrastive loss with respect to the attention logits at masked (future) positions. Use these gradients to construct a soft mask matrix where future positions receive small, gradient-proportional attention weights rather than being fully blocked. This lets the model tentatively explore bidirectional patterns where the gradient signal indicates it would be most beneficial, without sudden exposure to all future tokens. **Phase 2 (Linear Scheduler):** After the pre-warmup stabilizes, apply a linear interpolation coefficient `alpha(t)` that transitions the mask from causal (`alpha=0`) to fully bidirectional (`alpha=1`) over the remaining training steps: `M(t) = (1 - alpha(t)) * M_causal + alpha(t) * M_bidirectional`. The combination of gradient-guided warmup followed by linear scheduling prevents the training instabilities that plague direct or scheduler-only transitions.

### Contrastive Framework for User Behaviors

The model processes heterogeneous user behavior sequences (e.g., payment events, browsing actions, merchant interactions tokenized into a flat sequence) through the modified decoder-only LLM. Mean pooling over the final hidden states produces the user embedding. Training uses a contrastive loss (InfoNCE) where positive pairs come from different time windows of the same user's behavior, and negatives are other users in the batch. This framework is architecture-agnostic — it works with any decoder-only backbone (Qwen, LLaMA, GPT-2, etc.).

## Step-by-Step Workflow

1. **Tokenize heterogeneous user behaviors into a flat sequence.** Map each behavioral event (e.g., `{type: "purchase", merchant: "coffee_shop", amount: 15.50, timestamp: ...}`) to a subsequence of tokens. Prepend a `[USER]` token and separate event types with special delimiter tokens. Truncate or sample to a fixed maximum length (e.g., 512 or 1024 tokens).

2. **Initialize from a pretrained decoder-only checkpoint.** Load a causal LLM (e.g., Qwen-2, LLaMA-3, or GPT-2) with its original causal attention mask intact. Add a mean-pooling head on top of the final hidden states to produce a fixed-dimension user embedding.

3. **Implement the three attention mask matrices.** Define `M_causal` (lower-triangular), `M_bidirectional` (all-ones), and the interpolated mask `M(t)`. Store these as buffers in the attention module so they can be swapped during training:
   ```python
   # Causal: token i attends only to tokens j <= i
   M_causal = torch.tril(torch.ones(seq_len, seq_len))
   # Bidirectional: full attention
   M_bidir = torch.ones(seq_len, seq_len)
   # Interpolated at step t
   M_t = (1 - alpha_t) * M_causal + alpha_t * M_bidir
   ```

4. **Implement the Gradient-Guided Soft Masking pre-warmup.** For the first `N_warmup` steps, compute a forward pass with the causal mask, then compute the gradient of the contrastive loss with respect to the attention logits at the masked (upper-triangular) positions. Normalize these gradients to `[0, epsilon]` (e.g., `epsilon=0.1`) and use them as soft attention weights for the future positions:
   ```python
   # During pre-warmup phase
   with torch.enable_grad():
       attn_logits.requires_grad_(True)
       loss = contrastive_loss(model(input, mask=M_causal))
       grad = torch.autograd.grad(loss, attn_logits)[0]

   # Soft mask: future positions get small gradient-proportional weights
   future_mask = (1 - M_causal)  # upper triangle
   grad_magnitude = grad.abs() * future_mask
   grad_normalized = epsilon * (grad_magnitude / (grad_magnitude.max() + 1e-8))
   M_soft = M_causal + grad_normalized
   ```

5. **Implement the linear scheduler for the transition phase.** After `N_warmup` steps, linearly increase `alpha` from 0 to 1 over `N_transition` steps. The model sees progressively more bidirectional attention:
   ```python
   if step < N_warmup:
       mask = M_soft  # gradient-guided
   elif step < N_warmup + N_transition:
       alpha = (step - N_warmup) / N_transition
       mask = (1 - alpha) * M_causal + alpha * M_bidir
   else:
       mask = M_bidir  # fully bidirectional
   ```

6. **Define the contrastive learning objective.** Use InfoNCE loss over user embedding pairs. Construct positives from different temporal windows of the same user's behavior, and use in-batch negatives:
   ```python
   def infonce_loss(z_i, z_j, temperature=0.07):
       z_i = F.normalize(z_i, dim=-1)
       z_j = F.normalize(z_j, dim=-1)
       logits = z_i @ z_j.T / temperature
       labels = torch.arange(z_i.size(0), device=z_i.device)
       return F.cross_entropy(logits, labels)
   ```

7. **Train with the combined schedule.** Use AdamW with linear warmup + cosine decay on the learning rate. Monitor the contrastive loss and gradient norms closely during the mask transition period. If gradient norms spike, reduce `epsilon` or extend `N_warmup`.

8. **Extract user embeddings via mean pooling.** At inference time, run the model with the fully bidirectional mask and mean-pool the last hidden layer across all sequence positions to get the user embedding vector.

9. **Evaluate on downstream tasks.** Probe the frozen user embeddings on classification tasks (purchase prediction, churn prediction, preference ranking, marketing sensitivity) using simple linear classifiers or k-NN retrieval.

10. **Tune the warmup/transition hyperparameters.** Key knobs: `N_warmup` (5-15% of total steps), `N_transition` (30-60% of total steps), `epsilon` (0.01-0.2), `temperature` (0.05-0.1). Longer warmup and lower epsilon produce more stable but slower transitions.

## Concrete Examples

**Example 1: Building a User Embedding Model from E-Commerce Logs**

User: "I have user clickstream data with page views, add-to-carts, and purchases. I want to build user embeddings using a small GPT-2 model with contrastive learning."

Approach:
1. Tokenize each event into `[EVENT_TYPE] [ITEM_ID] [CATEGORY] [TIMESTAMP_BIN]` subsequences
2. Concatenate events chronologically per user, truncate to 512 tokens
3. Load GPT-2 (124M), replace the LM head with a mean-pooling layer
4. Create positive pairs by splitting each user's history at a random midpoint (first half vs. second half)
5. Implement GGSM: 2K warmup steps with `epsilon=0.05`, then 10K linear transition steps
6. Train with InfoNCE loss, batch size 256, for 50K total steps

Output:
```python
class UserEncoder(nn.Module):
    def __init__(self, base_model_name="gpt2"):
        super().__init__()
        self.backbone = AutoModel.from_pretrained(base_model_name)
        self.embed_dim = self.backbone.config.hidden_size

    def forward(self, input_ids, attention_mask, custom_attn_mask=None):
        outputs = self.backbone(
            input_ids=input_ids,
            attention_mask=attention_mask,
            # Inject custom causal/bidirectional/soft mask
        )
        hidden = outputs.last_hidden_state  # (B, L, D)
        # Mean pool over non-padding positions
        mask_expanded = attention_mask.unsqueeze(-1).float()
        pooled = (hidden * mask_expanded).sum(1) / mask_expanded.sum(1)
        return F.normalize(pooled, dim=-1)
```

**Example 2: Stabilizing a Causal-to-Bidirectional Transition**

User: "I'm fine-tuning a LLaMA model for user embeddings but training collapses when I switch to bidirectional attention at step 5000. How do I fix this?"

Approach:
1. Diagnose: the abrupt mask switch causes gradient explosion because the model has never seen future-token attention patterns
2. Replace the hard switch with GGSM: add a gradient-guided pre-warmup phase starting at step 4000
3. Compute gradients of the loss w.r.t. attention logits at upper-triangular (future) positions
4. Use normalized gradient magnitudes as soft weights (epsilon=0.05) for those positions
5. After 1000 warmup steps, begin the linear transition over 4000 steps (steps 5000-9000)
6. Monitor gradient norm — it should remain within 2x of the pre-transition baseline

Output:
```python
class GGSMScheduler:
    def __init__(self, warmup_start, warmup_steps, transition_steps, epsilon=0.05):
        self.warmup_start = warmup_start
        self.warmup_end = warmup_start + warmup_steps
        self.transition_end = self.warmup_end + transition_steps
        self.epsilon = epsilon

    def get_mask(self, step, seq_len, grad_info=None):
        causal = torch.tril(torch.ones(seq_len, seq_len))
        bidir = torch.ones(seq_len, seq_len)

        if step < self.warmup_start:
            return causal
        elif step < self.warmup_end:
            # Gradient-guided soft masking
            if grad_info is not None:
                future = (1 - causal)
                soft = grad_info.abs() * future
                soft = self.epsilon * (soft / (soft.max() + 1e-8))
                return causal + soft
            return causal
        elif step < self.transition_end:
            alpha = (step - self.warmup_end) / (self.transition_end - self.warmup_end)
            return (1 - alpha) * causal + alpha * bidir
        else:
            return bidir
```

**Example 3: Evaluating User Embeddings on Downstream Tasks**

User: "I've trained user embeddings with GGSM. How do I evaluate them on preference prediction and marketing sensitivity tasks?"

Approach:
1. Freeze the user encoder and extract embeddings for all users in the evaluation set
2. For preference prediction: train a logistic regression on (user_embedding, item_embedding) dot products to predict click/purchase
3. For marketing sensitivity: train a linear classifier on user embeddings to predict whether a user responds to a specific campaign
4. Compare against baselines: causal-only embeddings, hybrid-mask embeddings, and scheduler-only (no GGSM) embeddings
5. Report AUC-ROC and accuracy; GGSM bidirectional should outperform by 1-3% over causal baselines

Output:
```python
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import roc_auc_score

# Extract frozen embeddings
with torch.no_grad():
    user_embeds = encoder(behavior_tokens, mask=bidir_mask)

# Preference prediction probe
clf = LogisticRegression(max_iter=1000)
clf.fit(user_embeds_train, preference_labels_train)
preds = clf.predict_proba(user_embeds_test)[:, 1]
auc = roc_auc_score(preference_labels_test, preds)
print(f"Preference AUC: {auc:.4f}")

# Marketing sensitivity probe
clf_mkt = LogisticRegression(max_iter=1000)
clf_mkt.fit(user_embeds_train, campaign_response_train)
preds_mkt = clf_mkt.predict_proba(user_embeds_test)[:, 1]
auc_mkt = roc_auc_score(campaign_response_test, preds_mkt)
print(f"Marketing Sensitivity AUC: {auc_mkt:.4f}")
```

## Best Practices

- **Do:** Start with the causal mask fully intact and only begin the GGSM warmup after the base contrastive loss has stabilized (typically a few thousand steps). Premature mask transitions waste the pretrained causal structure.
- **Do:** Use mean pooling over all sequence positions rather than last-token pooling. Bidirectional attention distributes information across all positions, so last-token pooling discards useful signal.
- **Do:** Monitor gradient norms at masked positions during the pre-warmup phase. If they exceed 10x the unmasked gradient norm, reduce `epsilon`.
- **Do:** Construct contrastive positive pairs from temporally distinct windows of the same user's behavior — this forces the model to capture stable user traits rather than transient session patterns.
- **Avoid:** Switching from causal to bidirectional attention in a single step. This is the primary failure mode the paper identifies — it causes training collapse.
- **Avoid:** Setting `epsilon` too high (>0.3) during the gradient-guided phase. The soft mask should provide gentle exposure to future attention, not overwhelm the causal structure.
- **Avoid:** Using the hybrid mask (causal on behavioral tokens, bidirectional on a CLS token) as a compromise — the paper shows this consistently underperforms the full GGSM bidirectional approach.

## Error Handling

- **Training loss spikes during mask transition:** Reduce `epsilon` by half, extend `N_warmup` by 50%, and add gradient clipping (max_norm=1.0). If spikes persist, checkpoint the model at pre-warmup and restart the transition with a slower schedule.
- **Gradient-guided mask produces near-zero values everywhere:** The contrastive loss may be too low for meaningful gradients at masked positions. Increase the learning rate temporarily during warmup, or use a higher temperature in InfoNCE to sharpen gradients.
- **User embeddings collapse to a single point (mode collapse):** Increase the number of in-batch negatives, add a small L2 regularization to the pooled embeddings, or use a momentum encoder (MoCo-style) to stabilize the contrastive target.
- **Out-of-memory during gradient computation for soft mask:** The gradient w.r.t. attention logits can be large for long sequences. Compute the gradient-guided mask on a subset of attention heads or use gradient checkpointing.
- **Downstream task performance doesn't improve over causal baseline:** Ensure the transition completes fully to bidirectional attention. Partially transitioned models (alpha < 0.8) may not capture enough bidirectional context to outperform causal.

## Limitations

- The technique is designed for **user representation (embedding) tasks**, not for generative (text output) tasks. Bidirectional attention breaks autoregressive generation, so the resulting model cannot be used for text generation without switching the mask back.
- GGSM adds **computational overhead** during training: each warmup step requires an extra backward pass to compute gradients for the soft mask. Expect ~30-50% slower training during the warmup phase.
- The approach is validated on **behavioral sequence data** (transactions, clicks). Its effectiveness on other sequential user data like natural language posts or social media activity is plausible but unverified by the paper.
- **Small models (< 100M params)** may not benefit as much, since the pretrained causal attention patterns that GGSM preserves are less structured in small models.
- The paper evaluates on Alipay-scale industrial data. On **small datasets** (< 100K users), the contrastive learning framework may not provide enough signal, and simpler methods like mean-pooling pretrained token embeddings may suffice.

## Reference

**Paper:** [How Do Decoder-Only LLMs Perceive Users? Rethinking Attention Masking for User Representation Learning](https://arxiv.org/abs/2602.10622v1) (Yuan et al., 2026). Look for Section 3 (GGSM formulation and mask scheduling) and Section 5 (ablation over warmup length, epsilon, and transition speed on the 9 benchmarks).

**Code:** [https://github.com/JhCircle/Deepfind-GGSM](https://github.com/JhCircle/Deepfind-GGSM) (Apache-2.0)