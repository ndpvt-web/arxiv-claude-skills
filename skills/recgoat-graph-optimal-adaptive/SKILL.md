---
name: "recgoat-graph-optimal-adaptive"
description: "Build multimodal recommendation systems that align LLM semantic embeddings with collaborative filtering ID embeddings using graph attention networks, cross-modal contrastive learning, and optimal adaptive transport. Use when: 'align LLM embeddings with recommendation IDs', 'build multimodal recommender with text and image features', 'bridge semantic and collaborative representations', 'optimal transport for feature alignment', 'graph-based multimodal recommendation pipeline', 'dual semantic alignment for rec system'."
---

# RecGOAT: Graph Optimal Adaptive Transport for LLM-Enhanced Multimodal Recommendation

This skill enables Claude to build multimodal recommendation systems that solve the fundamental mismatch between LLM-derived semantic representations (text, vision) and sparse collaborative filtering ID embeddings. It implements the RecGOAT dual semantic alignment framework: graph attention networks enrich each modality with collaborative signals, then cross-modal contrastive learning aligns representations at the instance level while optimal adaptive transport aligns them at the distribution level. The result is a unified user/item representation that captures both rich semantics and personalized collaborative patterns.

## When to Use

- When the user wants to integrate LLM text/vision embeddings into a recommendation system but finds that naively concatenating or adding them degrades performance
- When building a multimodal recommender that must fuse item images, text descriptions, and user interaction history into a single embedding space
- When the user asks how to align distributions of embeddings from different modalities (e.g., CLIP features vs. learned ID embeddings)
- When implementing graph-based collaborative filtering that operates on LLM-derived node features rather than random initializations
- When the user needs to add optimal transport-based distribution alignment to any cross-modal learning pipeline
- When scaling a multimodal recommender from research datasets (Baby, Sports, Electronics) to production ad or content platforms

## Key Technique

**The Core Problem.** LLM embeddings live in a semantic space optimized for language understanding, while recommendation ID embeddings live in a collaborative space shaped by interaction patterns. Directly fusing them (concatenation, addition, gating) fails because the geometric structures of these spaces are incompatible. Items that are semantically similar (e.g., two red shirts) may have very different interaction patterns, and vice versa.

**Dual-Granularity Alignment.** RecGOAT solves this with two complementary alignment mechanisms. First, *instance-level alignment* via cross-modal contrastive learning (CMCL) pulls together the text, visual, and ID representations of the same item while pushing apart representations of different items, using InfoNCE loss across all modality pairs (ID-text, ID-visual, text-visual). Second, *distribution-level alignment* via optimal adaptive transport (OAT) minimizes the 1-Wasserstein distance between the modality embedding distribution and the ID embedding distribution across a batch. The Sinkhorn algorithm computes an initial transport plan, then a learnable residual matrix adapts it to the recommendation task. Transported features `Z_hat = Z_modal * T` are geometrically remapped into ID space.

**Graph-Enriched Modality Representations.** Before alignment, each modality is enriched with collaborative structure. Item-item graphs (k-NN on LLM features) use multi-head GAT to propagate modality-specific signals. User-item graphs use LightGCN-style propagation with rating-weighted attention. User-user graphs capture peer preferences from LLM-inferred user profiles. This ensures that the modality embeddings already carry collaborative signal before the alignment step, making the transport problem easier.

## Step-by-Step Workflow

1. **Extract LLM representations for items.** Encode item text descriptions through a text embedding model (e.g., Qwen3-Embedding or Sentence-BERT) to get text features `X_text`, and extract visual features from product images through a vision-language model (e.g., LLaVA, CLIP) to get `X_visual`. Store these as fixed feature matrices of shape `[num_items, embed_dim]`.

2. **Generate LLM user profiles.** For each user, construct a prompt containing their interaction history (item titles, categories, ratings). Pass this through a reasoning LLM (e.g., QwQ-32B, GPT-4) to generate a structured preference summary. Encode the summary through the same text embedding model to get user text features `U_text`.

3. **Build modality-specific item-item graphs.** For each modality (text, visual), compute k-nearest neighbors (k=10 typical) on the LLM feature vectors using cosine similarity. This produces adjacency matrices for text-item and visual-item graphs.

4. **Apply multi-head graph attention on item-item graphs.** Implement GAT layers that propagate features within each modality graph:
   ```python
   # For each modality m in {text, visual}:
   z_i_m = concat([
       sigma(sum(alpha_ij * W_h @ x_j_m for j in neighbors(i)))
       for h in range(num_heads)
   ])
   ```
   Use 2 GAT layers with 4 attention heads. This enriches item features with modality-specific collaborative structure.

5. **Apply LightGCN propagation on the user-item bipartite graph.** Use the interaction matrix with rating-weighted edges. Propagate for 2-3 layers, aggregating user and item representations from both sides:
   ```python
   E_u_next = sum(r_ui / sqrt(|N_u| * |N_i|) * E_i for i in user_items)
   E_i_next = sum(r_ui / sqrt(|N_u| * |N_i|) * E_u for u in item_users)
   ```
   Initialize item nodes with the GAT-enriched modality features and user nodes with LLM user profiles.

6. **Implement cross-modal contrastive learning (CMCL) for instance-level alignment.** For each modality pair (ID-text, ID-visual, text-visual), compute InfoNCE loss within each training batch:
   ```python
   def cmcl_loss(z_a, z_b, temperature=0.2):
       sim = cosine_similarity(z_a, z_b) / temperature
       labels = torch.arange(len(z_a))
       return F.cross_entropy(sim, labels)

   L_cmcl = cmcl_loss(z_id, z_text) + cmcl_loss(z_id, z_visual) + cmcl_loss(z_text, z_visual)
   ```

7. **Implement optimal adaptive transport (OAT) for distribution-level alignment.** For each modality, compute the transport plan from modality space to ID space:
   ```python
   def compute_oat(Z_modal, Z_id, sinkhorn_iters=20, epsilon=0.05):
       # Cost matrix: pairwise L1 distance, normalized
       C = torch.cdist(Z_modal, Z_id, p=1) / Z_modal.shape[0]
       # Sinkhorn iterations for initial plan T0
       T0 = sinkhorn(C, epsilon, sinkhorn_iters)
       # Learnable residual (initialized to zero)
       T_residual = nn.Parameter(torch.zeros_like(T0))
       T = T0 + T_residual
       # Transport features into ID space
       Z_aligned = Z_modal @ T
       # Loss: Wasserstein distance
       W1 = (T * C).sum()
       return Z_aligned, W1
   ```

8. **Fuse aligned representations into unified embeddings.** Combine the transported modality features with ID embeddings using learned weights:
   ```python
   Z_unified = gamma_t * Z_text_aligned + gamma_v * Z_visual_aligned + (1 - gamma_t - gamma_v) * Z_id
   ```
   Initialize `gamma_t = gamma_v = 0.1` so ID embeddings dominate early in training.

9. **Train with the combined objective.** Combine BPR ranking loss, CMCL loss, and Wasserstein distance:
   ```python
   L_total = L_bpr + lambda_cmcl * L_cmcl + lambda_oat * (W1_text + W1_visual)
   ```
   Use `lambda_cmcl=0.1`, `lambda_oat=0.01` as starting points. Train with Adam, lr=1e-3, batch size 2048.

10. **Evaluate and tune.** Measure Recall@10, Recall@20, NDCG@10, NDCG@20. Ablate by removing CMCL (expect ~3-5% drop) and OAT (expect ~2-4% drop) to verify both alignment stages contribute. Check that the unified embeddings outperform any single-modality embeddings.

## Concrete Examples

**Example 1: E-commerce product recommendation with text + images**

User: "I have an Amazon product dataset with item titles, descriptions, images, and user purchase history. I want to build a recommender that uses both text and image features from a pretrained LLM alongside collaborative filtering."

Approach:
1. Encode item text (title + description) with `sentence-transformers/all-MiniLM-L6-v2` to get 384-dim text features
2. Encode item images with CLIP ViT-B/32 to get 512-dim visual features; project both to 256-dim shared space
3. Build item-item k-NN graphs (k=10) separately for text and visual features
4. Apply 2-layer GAT (4 heads, 64-dim per head) on each graph to get enriched item features
5. Build user-item bipartite graph from purchase history; run 3-layer LightGCN
6. Apply CMCL across (ID, text), (ID, visual), (text, visual) pairs with temperature=0.2
7. Run Sinkhorn OAT (20 iterations, epsilon=0.05) to transport text and visual features to ID space
8. Fuse: `Z = 0.1*Z_text_aligned + 0.1*Z_visual_aligned + 0.8*Z_id`
9. Train with BPR loss + 0.1*CMCL + 0.01*OAT for 200 epochs

Output: A model that produces 256-dim unified user/item embeddings. At inference, compute dot-product scores between user and candidate item embeddings. Expect 3-8% improvement in Recall@10 over ID-only or naive feature concatenation baselines.

**Example 2: Adding optimal transport alignment to an existing multimodal recommender**

User: "I already have a recommender that uses CLIP embeddings concatenated with ID embeddings. Performance is mediocre. How can I improve the feature fusion?"

Approach:
1. Replace concatenation with the dual alignment pipeline
2. Keep existing CLIP features as `Z_visual` and any text features as `Z_text`
3. Add CMCL loss between each modality pair to pull same-item representations together:
   ```python
   # Add to existing training loop
   sim_matrix = F.cosine_similarity(z_clip.unsqueeze(1), z_id.unsqueeze(0), dim=-1)
   cmcl = F.cross_entropy(sim_matrix / 0.2, torch.arange(batch_size))
   ```
4. Add OAT module to transport CLIP features into ID space before fusion:
   ```python
   cost = torch.cdist(z_clip, z_id, p=1) / batch_size
   T = sinkhorn(cost, epsilon=0.05, iters=20)
   T = T + self.residual_transport  # learnable nn.Parameter
   z_clip_aligned = z_clip @ T
   ```
5. Replace `cat(z_clip, z_id)` with `0.15*z_clip_aligned + 0.85*z_id`

Output: Drop-in improvement to existing pipeline. The OAT ensures CLIP's semantic geometry is remapped to match collaborative filtering geometry rather than forcing the MLP to learn this mapping implicitly.

**Example 3: Building user-user collaborative graphs from LLM-generated profiles**

User: "I want to create a user similarity graph based on LLM-inferred preferences rather than just co-purchase overlap."

Approach:
1. For each user, compose a prompt with their recent 20 interactions:
   ```
   Based on these purchases: [item1: Red Running Shoes, item2: Yoga Mat, ...],
   summarize this user's preferences in 3 sentences covering: preferred categories,
   price sensitivity, and style preferences.
   ```
2. Run through GPT-4 or similar to get preference summaries
3. Encode summaries with a text embedding model to get `U_text` vectors
4. Build k-NN graph (k=15) on `U_text` using cosine similarity
5. Apply 2-layer GAT to propagate user features through the similarity graph
6. Use the enriched user features as initialization for the user side of the user-item graph

Output: A user-user graph where edges reflect semantic preference similarity rather than behavioral overlap. Users who have never interacted with the same items but have similar tastes (e.g., both prefer minimalist outdoor gear) become connected.

## Best Practices

- **Do:** Always enrich modality features with graph structure *before* alignment. Raw LLM embeddings lack collaborative signal, and aligning them directly to ID space is harder and less effective.
- **Do:** Use both CMCL and OAT together. CMCL handles instance-level pairing (same item, different modalities), while OAT handles distribution-level geometry (overall shape of the embedding cloud). They are complementary, not redundant.
- **Do:** Initialize fusion weights heavily toward ID embeddings (0.8+ for ID). Collaborative signal is the backbone of recommendation; modality features refine it.
- **Do:** Normalize features before computing the OAT cost matrix. Unnormalized features cause the transport plan to be dominated by scale differences rather than geometric structure.
- **Avoid:** Running Sinkhorn for too many iterations (>50) or with too small epsilon (<0.01). This causes numerical instability. Start with 20 iterations and epsilon=0.05.
- **Avoid:** Using OAT with very small batch sizes (<256). The transport plan estimates distribution alignment from a batch; small batches give noisy estimates. Use batch size 1024-2048.
- **Avoid:** Applying this framework when you lack multimodal item data. If items have only IDs and interaction data, standard collaborative filtering is more appropriate.

## Error Handling

- **Sinkhorn divergence:** If transport plan values explode to NaN, increase epsilon (entropy regularization) or reduce iterations. Add `torch.clamp` on intermediate values. Log-domain Sinkhorn is more numerically stable.
- **Contrastive loss collapse:** If CMCL loss drops to near-zero quickly, the temperature is too high. Decrease from 0.2 to 0.07. Also verify that negative samples within the batch are genuinely different items.
- **OOM on large graphs:** For datasets with >100K items, use mini-batch graph sampling (e.g., GraphSAINT or neighbor sampling) rather than full-graph GAT. The k-NN graph construction can use approximate nearest neighbors (FAISS, Annoy).
- **Modality missing for some items:** If some items lack images or text, use a learned default embedding (not zero vector) and mask those items from the CMCL loss. The OAT can still operate on the available subset.
- **Transport plan not learning:** If the residual transport matrix stays near zero, increase `lambda_oat` or use a larger learning rate specifically for the residual parameters.

## Limitations

- Requires precomputed LLM embeddings for all items and users, which is expensive for large catalogs (>1M items). The LLM inference is a one-time cost but can be substantial.
- The Sinkhorn OAT operates per-batch, so distribution alignment quality depends on batch size. Very sparse datasets with few interactions per user may not provide enough signal for meaningful transport plans.
- The framework assumes that all modalities share the same embedding dimensionality after projection. Extremely high-dimensional LLM outputs (e.g., 4096-dim) should be projected down before alignment.
- Cold-start items with no interactions cannot benefit from the collaborative graph enrichment. They fall back to pure LLM semantic features, which is better than nothing but worse than the full pipeline.
- The theoretical guarantees (consistency, comprehensiveness) hold under assumptions about feature smoothness and graph connectivity that may not hold for all real-world datasets.

## Reference

**Paper:** [RecGOAT: Graph Optimal Adaptive Transport for LLM-Enhanced Multimodal Recommendation with Dual Semantic Alignment](https://arxiv.org/abs/2602.00682v1) -- Li et al., 2026. Focus on Section 3 (methodology) for the graph enrichment and dual alignment framework, Section 4 for theoretical guarantees, and Section 5.4 for ablation studies showing the contribution of each component.