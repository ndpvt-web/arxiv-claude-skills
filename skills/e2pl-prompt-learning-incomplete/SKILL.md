---
name: "e2pl-prompt-learning-incomplete"
description: "Design prompt-learning systems for incremental multi-view multi-label classification with missing data. Use when: 'handle missing views in multi-label classification', 'incremental learning with incomplete multimodal data', 'tensor decomposition for prompt efficiency', 'class-incremental multi-view learning', 'reduce exponential prompt parameters to linear', 'robust multi-label prediction with missing modalities'."
---

# E2PL: Prompt Learning for Incomplete Multi-View Multi-Label Incremental Classification

This skill enables Claude to design and implement prompt-learning systems that handle two simultaneous challenges: **missing views** (some data modalities are unavailable for some samples) and **class-incremental learning** (new label classes arrive over time). The core technique from E2PL uses task-tailored prompts for incremental adaptation, missing-aware prompts indexed by binary view-availability masks, and tensor-train decomposition to collapse exponential prompt storage into linear cost. This is directly applicable to any multimodal classification pipeline where inputs are heterogeneously incomplete and the label space grows.

## When to Use

- When building a multi-view or multimodal classifier where some modalities are randomly missing at both train and test time
- When the classification system must learn new classes incrementally without retraining from scratch (class-incremental learning)
- When a naive approach to handling all 2^n missing-view patterns would explode parameter count exponentially
- When you need prompt-based adaptation on a frozen pretrained backbone to avoid catastrophic forgetting
- When combining multi-label prediction (each sample has multiple tags) with incomplete multimodal inputs
- When designing contrastive learning objectives that account for which views two samples share

## Key Technique

**The Problem.** In multi-view multi-label class-incremental learning (IMvMLCIL), a model receives data from n views (e.g., GIST, HSV, RGB features of images), but any subset of views can be missing for any sample. The model must predict across all accumulated label classes, including newly introduced ones. A naive prompt-per-pattern approach requires 2^n separate prompts -- infeasible as views grow.

**The E2PL Solution.** E2PL attaches two types of learnable prompts to a frozen transformer backbone: (1) **Task-Tailored Prompts (TTPs)** -- one per incremental session, injected into the transformer's key/value attention vectors, where only the current session's prompt trains while previous ones freeze; (2) **Missing-Aware Prompts (MAPs)** -- indexed by a binary view-indicator vector m in {0,1}^n that records which views are present. The MAP for pattern m is computed as P_M^m = A * beta_m^T, where A is a shared basis matrix and beta_m is produced by a **Tensor-Train (TT) decomposition**: beta_m = G_1(m_1) * G_2(m_2) * ... * G_n(m_n) * G_{n+1}. Each G_l is a small core tensor indexed by whether view l is present (0 or 1). This reduces storage from O(2^n) to O(n), making it feasible for dozens of views.

**Dynamic Contrastive Learning.** To teach the model that samples sharing views should have similar representations, E2PL computes a view-overlap score s_ij = m_i^T * sigmoid(w) * m_j. Pairs with positive overlap are pulled together; non-overlapping pairs are pushed apart with margin alpha. The total loss is L = L_BCE + lambda * L_DCL.

## Step-by-Step Workflow

1. **Define the view structure.** Enumerate the n modalities/views in your data. Assign each a binary index position. For each sample, construct a view-indicator vector m in {0,1}^n where m_l = 1 if view l is available, 0 otherwise.

2. **Set up view-specific encoders.** For each view l, create a linear encoder f_l that maps raw features x_l to a shared embedding dimension d. Impute missing views with zero vectors before encoding (the prompts handle the actual adaptation).

3. **Initialize the frozen backbone.** Load a pretrained transformer and freeze all its parameters. All adaptation happens through prompt injection only -- this prevents catastrophic forgetting.

4. **Build Task-Tailored Prompts (TTPs).** For each incremental session t, allocate a learnable prompt matrix P_T^t of shape (L_p x d) where L_p is prompt length. Split each prompt into key and value components. Inject these into the transformer's multi-head attention K and V matrices at each layer.

5. **Build the Efficient Prototype Tensorization (EPT) module for MAPs.** Initialize: (a) a shared basis matrix A of shape (d x k) where k is the factor dimension (e.g., k=4); (b) n+1 TT-core tensors G_1...G_{n+1} where each G_l has shape (r_{l-1} x 2 x r_l) with TT-ranks r (e.g., r=2). To get the MAP for pattern m: compute beta_m = G_1(m_1) * G_2(m_2) * ... * G_n(m_n) * G_{n+1}, then P_M^m = A * beta_m^T.

6. **Combine prompts for each sample.** For sample i in session t: retrieve MAP P_M^{m_i} using its view-indicator, retrieve TTP P_T^t for the current task, concatenate or add them, then split into key/value components and inject into the transformer alongside the encoded (and zero-imputed) multi-view features.

7. **Implement the classification head.** Use a per-session linear classifier that outputs logits for that session's label classes. At inference, run all T session prompts and concatenate logits: Y_hat = [y^1; y^2; ...; y^T]. Apply sigmoid for multi-label prediction.

8. **Implement Dynamic Contrastive Learning.** For a mini-batch, compute pairwise view-overlap scores s_ij. Construct positive pairs (s_ij > 0) and negative pairs (s_ij = 0). Apply contrastive loss with margin alpha=1 to push apart non-overlapping pattern embeddings.

9. **Train with combined loss.** Optimize L_total = L_BCE + lambda * L_DCL (lambda ~ 0.001). Only update the current session's TTP and the shared EPT module. Freeze all prior TTPs.

10. **Evaluate incrementally.** After each session, test on all accumulated classes. Report mean average precision (mAP) averaged across sessions and on the final session to measure forgetting.

## Concrete Examples

**Example 1: Image annotation with 6 feature types and growing tag vocabulary**

User: "I have an image dataset with 6 feature views (GIST, HSV, DenseHue, DenseSift, RGB, LAB) but 30% of views are randomly missing per image. New annotation tags are added quarterly. Build a classifier."

Approach:
1. Construct view-indicator vectors m in {0,1}^6 for each image from the availability metadata.
2. Create 6 linear encoders mapping each feature type to d=128 dimensions. Zero-impute missing views.
3. Load a frozen ViT-Small backbone. Initialize TTP prompts (one per quarter/session, shape 5x128).
4. Build EPT module: shared basis A (128x4), 7 TT-cores with rank r=2. For 6 views, this stores ~56 parameters instead of 2^6=64 full prompts.
5. For each image, compute beta_m via TT-core chain, form MAP, inject with current TTP into the transformer.
6. Train with BCE + 0.001 * DCL loss. Freeze previous TTPs each quarter.

Output:
```python
class EPTModule(nn.Module):
    def __init__(self, n_views=6, d=128, k=4, tt_rank=2):
        super().__init__()
        self.A = nn.Parameter(torch.randn(d, k))  # shared basis
        # TT-cores: G_l has shape (r_{l-1}, 2, r_l)
        self.cores = nn.ParameterList([
            nn.Parameter(torch.randn(1 if l == 0 else tt_rank, 2, tt_rank))
            for l in range(n_views)
        ])
        # Final core maps to factor dimension k
        self.core_final = nn.Parameter(torch.randn(tt_rank, k))

    def forward(self, m):
        """m: (batch, n_views) binary view-indicator."""
        batch_size = m.shape[0]
        # Contract TT-cores along the chain
        result = None
        for l, core in enumerate(self.cores):
            # Index into core by m[:, l] (0 or 1)
            idx = m[:, l].long()                    # (batch,)
            slice_l = core[:, idx, :]               # (r_in, batch, r_out)
            slice_l = slice_l.permute(1, 0, 2)      # (batch, r_in, r_out)
            if result is None:
                result = slice_l                     # (batch, 1, r)
            else:
                result = torch.bmm(result, slice_l)  # (batch, 1, r)
        beta = torch.bmm(result, self.core_final.unsqueeze(0).expand(batch_size, -1, -1))
        beta = beta.squeeze(1)                       # (batch, k)
        prompt = torch.mm(beta, self.A.T)            # (batch, d)
        return prompt
```

**Example 2: Multimodal product classification with incremental categories**

User: "Our e-commerce platform has product data from 4 sources (text description, image, user reviews, structured attributes). Sources are often missing. New product categories are added monthly. How do I avoid retraining everything?"

Approach:
1. Define n=4 views. Each product gets m in {0,1}^4 based on available sources.
2. Use modality-specific encoders (BERT for text/reviews, ResNet for images, MLP for attributes) projecting to d=256.
3. Freeze a shared transformer fusion backbone after initial pretraining.
4. Allocate one TTP per month. Build EPT with 5 TT-cores (4 views + final), rank r=2, k=4. This needs ~40 parameters instead of 2^4=16 full prompt vectors.
5. Monthly: train only the new TTP + EPT on new categories. Prior TTPs stay frozen.

Output:
```
Parameter comparison for 4 views:
- Naive MAP (one prompt per pattern):  2^4 * d = 16 * 256 = 4,096 params
- EPT (TT-decomposed):               4 cores * (1*2*2) + final(2*4) + A(256*4)
                                     = 16 + 8 + 1024 = 1,048 params (74% reduction)

For 10 views:
- Naive MAP:  2^10 * 256 = 262,144 params
- EPT:        10 * 4 + 8 + 1024 = 1,072 params (99.6% reduction)
```

**Example 3: Designing the dynamic contrastive learning loss**

User: "How do I implement the contrastive loss that accounts for shared vs. disjoint view patterns?"

Approach:
1. Compute pairwise view-overlap using learnable weights w.
2. Classify pairs as positive (shared views) or negative (no overlap).
3. Apply margin-based contrastive loss.

Output:
```python
class DynamicContrastiveLoss(nn.Module):
    def __init__(self, n_views, alpha=1.0):
        super().__init__()
        self.w = nn.Parameter(torch.zeros(n_views))  # learnable view weights
        self.alpha = alpha

    def forward(self, embeddings, view_masks):
        """
        embeddings: (B, d) - MAP embeddings for each sample
        view_masks: (B, n_views) - binary view indicators
        """
        w_sigmoid = torch.sigmoid(self.w)  # (n_views,)
        # Weighted view overlap: (B, B)
        weighted_masks = view_masks * w_sigmoid.unsqueeze(0)  # (B, n_views)
        overlap = torch.mm(weighted_masks, view_masks.T)       # (B, B)

        # Pairwise distances
        dists = torch.cdist(embeddings, embeddings, p=2)       # (B, B)

        # Positive pairs: overlap > 0, pull together
        pos_mask = (overlap > 0).float()
        pos_loss = (pos_mask * dists.pow(2)).sum() / pos_mask.sum().clamp(min=1)

        # Negative pairs: overlap == 0, push apart with margin
        neg_mask = (overlap == 0).float()
        neg_loss = (neg_mask * F.relu(self.alpha - dists).pow(2)).sum() / neg_mask.sum().clamp(min=1)

        return pos_loss + neg_loss
```

## Best Practices

- **Do:** Zero-impute missing views before encoding -- the missing-aware prompts handle adaptation, not the raw input.
- **Do:** Keep TT-ranks small (r=2 is sufficient in practice). The decomposition is meant to be aggressive; higher ranks waste parameters without meaningful accuracy gains.
- **Do:** Freeze all previous task prompts when training a new session. This is the key mechanism preventing catastrophic forgetting.
- **Do:** Use the view-indicator vector m directly as the TT-core index -- this gives you O(n) computation per sample instead of maintaining a lookup table.
- **Avoid:** Training the frozen backbone parameters. The entire point is parameter-efficient adaptation through prompts only.
- **Avoid:** Using large factor dimension k. The paper uses k=4; going above 8 rarely helps and increases the shared basis matrix A.
- **Avoid:** Skipping the contrastive loss. Without DCL, the model treats all missing-view patterns as independent, missing the structural similarity between patterns that share views.

## Error Handling

- **All views missing for a sample:** The TT-core chain with all-zero indices still produces a valid beta vector (it selects the m_l=0 slice of each core). The output degenerates to a "no information" prompt. Flag these samples in logs and consider excluding them from contrastive loss computation.
- **Imbalanced missing rates across views:** If one view is rarely missing, its TT-core slice for m_l=0 gets few gradient updates. Add a small regularization term or oversample rare missing patterns during training.
- **New session has very few new classes:** The TTP for that session may overfit. Use a shorter prompt length L_p for sessions with few classes, or share a portion of the prompt with adjacent sessions.
- **NaN in TT-core chain:** Long chains of small matrix multiplications can underflow. Use float32 (not float16) for the EPT module and add gradient clipping (max_norm=1.0).

## Limitations

- **Assumes fixed view set.** The TT decomposition is structured around n known views. If a new modality is added, the entire EPT module must be restructured (add a new core tensor and retrain).
- **Binary view availability only.** Views are either fully present or fully absent. Partial views (e.g., a corrupted image) are not modeled -- they must be thresholded to present/absent.
- **Prompt injection requires transformer backbone.** The technique assumes K/V prompt injection into a transformer. It does not directly apply to CNN or GNN backbones without architectural adaptation.
- **Quadratic contrastive loss.** The DCL loss computes all-pairs distances in a batch, scaling as O(B^2). For large batch sizes, use random negative sampling instead.
- **Linear in n, but n must be moderate.** While exponential is reduced to linear, very large n (>50 views) still produces long TT-core chains with potential numerical issues.

## Reference

**Paper:** [E2PL: Effective and Efficient Prompt Learning for Incomplete Multi-view Multi-Label Class Incremental Learning](https://arxiv.org/abs/2601.17076v1) (Chen et al., 2026). Focus on Section 3 for the EPT tensor-train decomposition derivation and Section 4 for the dynamic contrastive learning formulation. Key result: 86% parameter reduction (0.026M vs 0.193M) with 3+ mAP improvement on MIRFLICKR at 30% missing views.