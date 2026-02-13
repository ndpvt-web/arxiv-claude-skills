---
name: "one-size-many-fits"
description: "Build group-aware advertising image generation systems that align diverse user-segment click preferences instead of optimizing for a single average. Use when: 'generate ad images for different user segments', 'build group-aware CTR optimization', 'implement preference-aligned image generation', 'create user-group clustering for ad personalization', 'build a Group-DPO training pipeline', 'design multi-segment advertising creative systems'."
---

# One Size, Many Fits: Group-Wise Preference-Aligned Advertising Image Generation

This skill enables Claude to help build advertising image generation systems that serve *diverse user groups* rather than optimizing for a single aggregate metric. Based on the OSMF framework, the core insight is: users cluster into distinct preference groups per product, and a single multimodal model can be conditioned on group embeddings to generate tailored creatives for each segment. The technique combines dynamic user grouping (product-aware adaptive grouping), a group-conditioned multimodal LLM for prompt/image generation (G-MLLM), and a group-level variant of Direct Preference Optimization (Group-DPO) to align outputs with each segment's click behavior.

## When to Use

- When the user wants to generate personalized ad creatives for different audience segments rather than one generic image per product
- When building a recommendation or creative generation system that needs to respect preference diversity across user clusters
- When implementing DPO-style preference alignment conditioned on group features (not just individual or global preferences)
- When constructing a user-grouping pipeline that clusters users by product-specific click behavior
- When designing a reward model that scores image-group compatibility rather than image quality alone
- When the user asks how to move beyond "one-size-fits-all" CTR optimization in ad tech

## Key Technique

**The Problem.** Standard ad image generation optimizes for aggregate CTR across all users. But different user segments (e.g., price-sensitive vs. brand-loyal, young vs. older demographics) prefer visually different creatives for the same product. A single optimized image suppresses minority-group preferences and leaves CTR on the table.

**OSMF's Three-Stage Solution.** First, *Product-Aware Adaptive Grouping (PAAG)* dynamically clusters users per product. User attributes are embedded via MLP, then cross-attended with product text features and product image features in sequence, producing product-specific user embeddings. K-means clustering (with silhouette-score-based K selection, capped at K=5) partitions users into groups. Each group is represented not by a single centroid but by percentile-sampled points (15th, 55th, 95th percentiles at ratios 2:3:5) to capture both core and peripheral preferences. Second, a *Group-aware MLLM (G-MLLM)* takes group embeddings as a prefix token alongside product image/text tokens and is pretrained on four tasks: group analysis (describe the group), behavioral prediction (predict who clicks), product comprehension (predict product from image), and prompt generation (write image-gen prompts for the group). Third, *Group-DPO* fine-tunes the G-MLLM using pairwise preference data per group. A frozen Group-aware Reward Model (GRM) converts (image, group, CTR) quadruples into winner/loser prompt pairs, and the standard DPO loss is applied conditioned on group embeddings: the model learns to prefer prompts that produce higher-CTR images *for that specific group*.

**Why It Works.** The group embedding acts as a lightweight conditioning signal that steers generation without separate models per segment. Group-DPO ensures alignment is segment-specific -- a prompt preferred by group A can be the loser for group B on the same product. The percentile-based group representation avoids collapsing diverse preferences into a single mean vector.

## Step-by-Step Workflow

1. **Define user attribute schema.** Identify categorical and numerical user features (age bracket, gender, purchase history tier, browsing categories, device type). Define product features (title text, main image, category, price tier). These form the inputs to the grouping module.

2. **Build the user embedding pipeline.** For each user, pass categorical attributes through embedding layers and concatenate with numerical features. Feed through a 2-layer MLP to produce a 128-dim user embedding vector. Implement this as a PyTorch `nn.Module` with configurable attribute vocabulary sizes.

3. **Implement product-aware cross-attention.** Create a cross-attention block where user embeddings attend to product text features (from a CLIP text encoder), producing product-text-aware user representations. Chain a second cross-attention block where these attend to product image features (from a ResNet or CLIP vision encoder). Output: product-specific user embedding per user.

4. **Cluster users per product with adaptive K.** For each product, collect all product-specific user embeddings. Run K-means for K in [2..5], compute silhouette scores, select K* that maximizes mean silhouette coefficient. Assign users to clusters. For products with insufficient users (<50), fall back to K=1 (no segmentation).

5. **Construct group representations via percentile sampling.** For each cluster, compute distances from centroid. Sample representative embeddings at the 15th percentile (2 samples), 55th percentile (3 samples), and 95th percentile (5 samples). Concatenate or pool these 10 vectors into a single group embedding. This captures both typical and edge-case preferences.

6. **Pretrain G-MLLM on four auxiliary tasks.** Using a LLaVA-style backbone, prepend the group embedding as a special token to the input sequence `[group_embed; image_tokens; text_tokens]`. Train on: (a) group description generation, (b) next-clicker prediction, (c) product title prediction from ad image, (d) group-tailored image prompt generation. Use cosine LR schedule (peak 2e-6), 10 epochs.

7. **Build the Group-aware Reward Model (GRM).** Train a binary classifier that takes (image_A, image_B, group_embedding) and predicts P(CTR_A > CTR_B | group). Use historical click logs to construct preference pairs: for each group, pair a high-CTR image with a low-CTR image for the same product. Train with binary cross-entropy.

8. **Generate Group-DPO training data.** For each product-group pair, use the pretrained G-MLLM to generate N candidate prompts. Render images from each prompt via the image generation backbone (e.g., ControlNet + Stable Diffusion). Score each image with the GRM relative to others. Select the top-scoring prompt as winner (y_w) and bottom-scoring as loser (y_l) per group.

9. **Fine-tune with Group-DPO.** Apply the DPO loss conditioned on group embeddings: `L = -log(sigma(beta * [log(pi/pi_ref)(y_w|s,G) - log(pi/pi_ref)(y_l|s,G)]))`. Use LoRA adapters on the G-MLLM (rank 16-64), LR 2e-5, 3 epochs. Keep the GRM frozen throughout.

10. **Deploy with group routing.** At inference time, classify the incoming user into their product-specific group using the trained PAAG module. Prepend the group embedding to the G-MLLM input. Generate a tailored image prompt, render the image, and serve it. Cache generated images per (product, group) pair for efficiency.

## Concrete Examples

**Example 1: Building the User Grouping Pipeline**

User: "I have a user table with age, gender, city_tier, and purchase_history, plus a product table with title and image_url. Help me build the product-aware adaptive grouping module."

Approach:
1. Define embedding layers for each categorical attribute and an MLP to fuse them into a 128-dim user vector.
2. Load a pretrained CLIP text encoder for product titles and a ResNet-50 for product images, projecting both to 128-dim.
3. Implement two `nn.MultiheadAttention` layers: user-to-text cross-attention, then output-to-image cross-attention.
4. For each product, gather user embeddings, run K-means with K in [2..5], select K* by silhouette score.
5. For each cluster, sample representative points at percentiles [15, 55, 95] with counts [2, 3, 5].

Output:
```python
class ProductAwareGrouping(nn.Module):
    def __init__(self, attr_vocabs, embed_dim=128, n_heads=4):
        super().__init__()
        self.attr_embeds = nn.ModuleDict({
            k: nn.Embedding(v, embed_dim) for k, v in attr_vocabs.items()
        })
        self.user_mlp = nn.Sequential(
            nn.Linear(embed_dim * len(attr_vocabs), embed_dim),
            nn.ReLU(),
            nn.Linear(embed_dim, embed_dim)
        )
        self.text_cross_attn = nn.MultiheadAttention(embed_dim, n_heads, batch_first=True)
        self.image_cross_attn = nn.MultiheadAttention(embed_dim, n_heads, batch_first=True)

    def forward(self, user_attrs, product_text_feats, product_image_feats):
        # user_attrs: dict of (batch, 1) tensors
        embeds = [self.attr_embeds[k](v) for k, v in user_attrs.items()]
        user_emb = self.user_mlp(torch.cat(embeds, dim=-1))  # (B, 128)
        user_emb = user_emb.unsqueeze(1)  # (B, 1, 128)
        # Cross-attend to product text
        text_aware, _ = self.text_cross_attn(user_emb, product_text_feats, product_text_feats)
        # Cross-attend to product image
        group_emb, _ = self.image_cross_attn(text_aware, product_image_feats, product_image_feats)
        return group_emb.squeeze(1)  # (B, 128)


def adaptive_cluster(embeddings, max_k=5, min_users=50):
    """Cluster product-specific user embeddings with silhouette-based K selection."""
    from sklearn.cluster import KMeans
    from sklearn.metrics import silhouette_score
    import numpy as np

    if len(embeddings) < min_users:
        return np.zeros(len(embeddings), dtype=int), 1

    best_k, best_score = 1, -1
    for k in range(2, min(max_k + 1, len(embeddings))):
        km = KMeans(n_clusters=k, n_init=10, random_state=42)
        labels = km.fit_predict(embeddings)
        score = silhouette_score(embeddings, labels)
        if score > best_score:
            best_k, best_score = k, score
            best_labels = labels
    return best_labels, best_k


def percentile_group_embedding(embeddings, centroid, percentiles=(15, 55, 95), counts=(2, 3, 5)):
    """Sample representative points at varying distances from centroid."""
    distances = np.linalg.norm(embeddings - centroid, axis=1)
    representatives = []
    for pct, count in zip(percentiles, counts):
        target_dist = np.percentile(distances, pct)
        closest_idx = np.argsort(np.abs(distances - target_dist))[:count]
        representatives.append(embeddings[closest_idx])
    return np.concatenate(representatives, axis=0).mean(axis=0)  # Pool to single vector
```

**Example 2: Implementing Group-DPO Training**

User: "I have a pretrained G-MLLM and a frozen reward model. Help me implement the Group-DPO fine-tuning loop."

Approach:
1. Load the pretrained G-MLLM as both `policy` (trainable with LoRA) and `reference` (frozen copy).
2. For each (product, group) pair, feed winner and loser prompts through both models to get log-probabilities.
3. Compute the DPO loss with group conditioning.
4. Update only the LoRA parameters.

Output:
```python
import torch
import torch.nn.functional as F
from peft import get_peft_model, LoraConfig

def setup_group_dpo(base_model, lora_rank=32):
    """Prepare policy and reference models for Group-DPO."""
    ref_model = copy.deepcopy(base_model).eval()
    for p in ref_model.parameters():
        p.requires_grad = False

    lora_config = LoraConfig(r=lora_rank, lora_alpha=64, target_modules=["q_proj", "v_proj"])
    policy_model = get_peft_model(base_model, lora_config)
    return policy_model, ref_model


def group_dpo_loss(policy_model, ref_model, batch, beta=0.1):
    """
    batch contains:
      - group_embeds: (B, embed_dim) group conditioning vectors
      - product_tokens: (B, seq_len) product image+text tokens
      - winner_ids: (B, gen_len) token ids of preferred prompts
      - loser_ids: (B, gen_len) token ids of dispreferred prompts
    """
    def get_log_probs(model, input_tokens, group_embeds, target_ids):
        # Prepend group embedding to input sequence
        logits = model(input_tokens, group_embeds=group_embeds).logits
        log_probs = F.log_softmax(logits, dim=-1)
        # Gather log probs for target tokens
        return torch.gather(log_probs[:, :-1], 2, target_ids[:, 1:].unsqueeze(-1)).squeeze(-1).sum(-1)

    pi_w = get_log_probs(policy_model, batch["product_tokens"], batch["group_embeds"], batch["winner_ids"])
    pi_l = get_log_probs(policy_model, batch["product_tokens"], batch["group_embeds"], batch["loser_ids"])
    ref_w = get_log_probs(ref_model, batch["product_tokens"], batch["group_embeds"], batch["winner_ids"])
    ref_l = get_log_probs(ref_model, batch["product_tokens"], batch["group_embeds"], batch["loser_ids"])

    # DPO loss: prefer winner over loser, conditioned on group
    reward_diff = beta * ((pi_w - ref_w) - (pi_l - ref_l))
    loss = -F.logsigmoid(reward_diff).mean()
    return loss
```

**Example 3: Designing the Group-Aware Reward Model**

User: "I need a reward model that scores (image, group) pairs so I can rank ad creatives per segment."

Approach:
1. Encode both images with a shared vision encoder.
2. Cross-attend image features with the group embedding.
3. Output P(image_A preferred over image_B | group).

Output:
```python
class GroupAwareRewardModel(nn.Module):
    def __init__(self, vision_encoder, embed_dim=128, n_heads=4):
        super().__init__()
        self.vision = vision_encoder  # Frozen ResNet/CLIP
        self.group_attn = nn.MultiheadAttention(embed_dim, n_heads, batch_first=True)
        self.score_head = nn.Sequential(
            nn.Linear(embed_dim * 2, embed_dim),
            nn.ReLU(),
            nn.Linear(embed_dim, 1)
        )

    def encode_with_group(self, image, group_embed):
        img_feat = self.vision(image).unsqueeze(1)            # (B, 1, D)
        group_feat = group_embed.unsqueeze(1)                  # (B, 1, D)
        attended, _ = self.group_attn(img_feat, group_feat, group_feat)
        return attended.squeeze(1)                             # (B, D)

    def forward(self, image_a, image_b, group_embed):
        feat_a = self.encode_with_group(image_a, group_embed)
        feat_b = self.encode_with_group(image_b, group_embed)
        combined = torch.cat([feat_a, feat_b], dim=-1)
        return torch.sigmoid(self.score_head(combined))        # P(A > B | group)
```

## Best Practices

- **Do:** Cap the maximum number of groups per product at 5. Beyond this, groups become too small for reliable CTR estimation and the generation cache explodes.
- **Do:** Use percentile-based group representations (15th/55th/95th) rather than simple centroids. Centroids collapse preference diversity within a group, which is the exact problem OSMF solves.
- **Do:** Freeze the reward model during Group-DPO training. A shifting reward signal destabilizes preference alignment and causes mode collapse.
- **Do:** Pretrain the G-MLLM on all four auxiliary tasks before Group-DPO. Skipping pretraining drops online CTR gains from +5.5% to +1.4%.
- **Avoid:** Using global (non-group-conditioned) DPO. Standard DPO averages preferences across groups, reproducing the one-size-fits-all problem the framework is designed to solve.
- **Avoid:** Clustering users without product conditioning. User segments that are meaningful for electronics may be irrelevant for apparel. Always cluster in the product-aware embedding space.

## Error Handling

- **Degenerate clusters:** If silhouette scores are negative for all K values on a product, fall back to K=1 (single group). This happens when user preferences are genuinely uniform for that product.
- **Cold-start products:** New products lack click history for grouping. Use category-level group templates: cluster users who clicked similar products in the same category, then transfer those group embeddings.
- **Imbalanced groups:** If one group contains >90% of users, the grouping is not capturing meaningful preference splits. Add minimum group size constraints (e.g., each group must have >= 5% of users) and re-cluster.
- **Reward model disagreement:** If the GRM's pairwise accuracy drops below 55% on a holdout set for specific product categories, retrain with category-specific data augmentation or fall back to aggregate CTR ranking.
- **LoRA instability during Group-DPO:** If loss oscillates, reduce beta (the DPO temperature) from 0.1 to 0.05, or lower the learning rate. Group-DPO with small groups is more sensitive to beta than standard DPO.

## Limitations

- Requires substantial click-log data (the paper uses 40M users across 2M products). For products with <50 exposures, group-level CTR estimates are unreliable.
- The framework assumes discrete, relatively stable user groups. For highly dynamic preference shifts (e.g., flash sales, trending items), group assignments may become stale and need frequent reclustering.
- Image generation quality is bounded by the underlying diffusion model (ControlNet + Stable Diffusion in the paper). Group-DPO improves *which* images are generated, not the rendering fidelity itself.
- The four-task pretraining requires labeled data for group descriptions and behavioral predictions, which may not be available outside large e-commerce platforms.
- Percentile-based group representations assume roughly unimodal clusters. If a cluster is itself multimodal, the 15th/55th/95th sampling may miss sub-clusters.

## Reference

**Paper:** [One Size, Many Fits: Aligning Diverse Group-Wise Click Preferences in Large-Scale Advertising Image Generation](https://arxiv.org/abs/2602.02033v1) (Lu et al., 2026). Key sections: Section 3.1 for PAAG architecture and percentile sampling, Section 3.3 for the Group-DPO loss formulation, and Table 3 for online CTR ablations showing each component's contribution.