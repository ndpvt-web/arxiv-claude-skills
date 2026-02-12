---
name: "recgoat-graph-optimal-adaptive"
description: "Build multimodal recommendation systems that align LLM semantic embeddings with collaborative filtering ID features using graph attention networks, contrastive learning, and optimal transport. Use when asked to: 'align LLM embeddings with recommendation IDs', 'build a multimodal recommender with graph attention', 'bridge semantic and collaborative representations', 'implement optimal transport for feature alignment', 'combine text/image embeddings with user-item interaction graphs', 'dual-granularity alignment for recommendation'."
---

# RecGOAT: Graph Optimal Adaptive Transport for LLM-Enhanced Multimodal Recommendation

This skill enables Claude to implement dual semantic alignment pipelines that bridge the representational gap between large language model embeddings (rich in general semantics) and recommendation system ID features (sparse, interaction-driven). The core technique from RecGOAT combines graph attention networks for enriching collaborative signals with a two-level alignment strategy: instance-level cross-modal contrastive learning and distribution-level optimal adaptive transport. This produces unified user/item representations that are both semantically meaningful and recommendation-effective.

## When to Use

- When building a multimodal recommendation system that ingests text, images, and interaction logs, and the user wants LLM-derived features to actually improve ranking rather than adding noise.
- When a user has pre-computed LLM embeddings (e.g., from a sentence transformer or GPT) for catalog items and needs to fuse them with collaborative filtering signals from a user-item interaction graph.
- When the user asks how to align two fundamentally different embedding spaces -- one from a language model, one from matrix factorization or graph-based CF -- without collapsing either's structure.
- When implementing a recommendation pipeline that must work at industrial scale (millions of users/items) and needs distribution-level alignment, not just pointwise matching.
- When the user wants to add contrastive learning between modalities (text vs. image vs. ID) in a recommender, combined with an optimal transport objective for global distributional coherence.
- When refactoring an existing recommender to incorporate LLM features and the naive concatenation/projection approach is degrading performance.

## Key Technique

**The core problem**: LLM embeddings live in a semantic space optimized for language understanding (cosine similarity reflects meaning similarity), while recommendation ID embeddings live in a collaborative space optimized for predicting interactions (proximity reflects co-occurrence patterns). Naively concatenating or projecting between these spaces destroys the structure of both. RecGOAT solves this with a principled two-level alignment.

**Graph-enriched collaborative semantics**: Before alignment, RecGOAT builds three relationship subgraphs -- item-item (co-purchased/co-viewed), user-item (interactions), and user-user (shared preferences) -- and propagates information through graph attention layers. This enriches the raw LLM embeddings with collaborative context. The attention mechanism learns which neighbors carry useful collaborative signal, weighting item-item similarity edges differently from user-item interaction edges. The output is a set of graph-enhanced multimodal representations that already encode some collaborative structure.

**Dual-granularity progressive alignment**: The alignment happens at two complementary levels. (1) **Instance-level via Cross-Modal Contrastive Learning (CMCL)**: For each item, the multimodal representation (text + image, graph-enhanced) and the ID embedding are treated as a positive pair; other items form negatives. A temperature-scaled InfoNCE loss pulls matched pairs together and pushes mismatched pairs apart. This ensures pointwise correspondence. (2) **Distribution-level via Optimal Adaptive Transport (OAT)**: Treating the set of multimodal embeddings and the set of ID embeddings as two probability distributions, OAT finds the minimum-cost transport plan between them using entropy-regularized Sinkhorn iterations. The adaptive component adjusts the transport cost matrix per training epoch based on alignment confidence, preventing premature locking of incorrect correspondences. Together, CMCL handles local alignment while OAT handles global distributional coherence -- the combination is provably stronger than either alone.

## Step-by-Step Workflow

1. **Prepare interaction data**: Load user-item interaction logs into a sparse interaction matrix. Each entry represents an observed interaction (click, purchase, rating). Store as a COO-format sparse tensor for efficient graph construction.

2. **Generate LLM embeddings for items and users**: For items, concatenate title + description + category and encode through a sentence transformer (e.g., `all-MiniLM-L6-v2` or domain-specific model). For users, aggregate their review text or profile data similarly. Store as dense tensors of shape `[num_items, llm_dim]` and `[num_users, llm_dim]`. These are frozen during training.

3. **Build the heterogeneous interaction graph**: Construct three edge sets: (a) item-item edges from co-interaction counts above a threshold, (b) user-item edges directly from the interaction matrix, (c) user-user edges from Jaccard similarity of interaction histories above a threshold. Represent as a single heterogeneous graph (PyG `HeteroData` or DGL `DGLGraph` with edge types).

4. **Implement the graph attention encoder**: Build a multi-layer GAT that operates on the heterogeneous graph. Each layer computes attention-weighted message passing per edge type, then aggregates across types. Input features are the frozen LLM embeddings projected to `hidden_dim` via a linear layer. Use 2-3 GAT layers with multi-head attention (4-8 heads). Output: graph-enhanced multimodal representations `H_mm` of shape `[num_nodes, hidden_dim]`.

5. **Maintain learnable ID embeddings**: Initialize a standard embedding table for user IDs and item IDs of shape `[num_users, hidden_dim]` and `[num_items, hidden_dim]`. These are trained end-to-end and represent the collaborative filtering signal. Call these `H_id`.

6. **Implement Cross-Modal Contrastive Learning (CMCL) loss**: For a mini-batch of items, compute the InfoNCE loss between `H_mm[items]` and `H_id[items]`. Normalize both to unit vectors. Similarity matrix `S = H_mm @ H_id.T / tau` where `tau` is temperature (0.07-0.2). Loss = `-log(exp(S[i,i]) / sum_j(exp(S[i,j])))` averaged over the batch. Apply symmetrically (both directions).

7. **Implement Optimal Adaptive Transport (OAT) loss**: For a mini-batch, treat rows of `H_mm` and `H_id` as two discrete distributions (uniform weights `1/N`). Compute cost matrix `C = pairwise_distance(H_mm, H_id)` (squared L2 or cosine). Run Sinkhorn iterations (20-50 steps) with entropic regularization `epsilon` (0.05-0.1) to get transport plan `P`. OAT loss = `sum(P * C)`. The adaptive component: scale `epsilon` inversely with training epoch (start large for exploration, shrink for precision).

8. **Combine losses and train**: Total loss = `L_bpr + lambda_1 * L_cmcl + lambda_2 * L_oat + lambda_3 * L_reg`. `L_bpr` is Bayesian Personalized Ranking loss on the final fused embeddings `H_final = H_mm + H_id` (or learned gating). Start with `lambda_1=0.1, lambda_2=0.01, lambda_3=1e-5`. Train with Adam, lr=1e-3, batch size 1024-2048, for 100-200 epochs.

9. **Inference**: For a target user, compute `H_final[user]`, then rank all items by dot product `H_final[user] @ H_final[items].T`. The fused embedding captures both semantic richness (from LLM + graph) and collaborative fit (from ID + BPR training).

10. **Evaluate**: Measure Recall@K, NDCG@K, and HR@K on held-out interactions (leave-one-out or temporal split). Compare against: (a) ID-only baseline (LightGCN), (b) naive LLM concatenation, (c) contrastive-only (no OAT), (d) OAT-only (no CMCL) to validate the dual alignment contribution.

## Concrete Examples

**Example 1: E-commerce product recommendation with text + image features**

User: "I have an Amazon product dataset with user purchase history, product titles, descriptions, and images. I want to build a recommender that uses LLM embeddings for the text and a pretrained vision model for images, but my current approach of concatenating these with collaborative filtering embeddings performs worse than CF alone."

Approach:
1. Encode product text via sentence transformer -> `text_emb [N_items, 384]`
2. Encode product images via CLIP ViT -> `img_emb [N_items, 512]`
3. Project both to shared dim: `mm_emb = Linear(cat(text_emb, img_emb)) -> [N_items, 256]`
4. Build co-purchase graph from interaction matrix (items co-purchased by 3+ users get an edge)
5. Run 2-layer GAT over heterogeneous graph with `mm_emb` as node features -> `H_mm`
6. Initialize ID embeddings `H_id [N_items, 256]` randomly
7. Train with `L_bpr + 0.1 * L_cmcl + 0.01 * L_oat`

Output:
```python
# Core training loop sketch
for epoch in range(200):
    epsilon = max(0.01, 0.1 * (1 - epoch / 200))  # Adaptive OAT regularization
    for batch_users, pos_items, neg_items in dataloader:
        h_mm = gat_encoder(graph, mm_features)           # Graph-enhanced multimodal
        h_id_u, h_id_i = id_embed_user(batch_users), id_embed_item(pos_items)

        # Fused representations for BPR
        h_user = h_mm[batch_users] + h_id_u
        h_pos = h_mm[pos_items] + h_id_i
        h_neg = h_mm[neg_items] + id_embed_item(neg_items)
        loss_bpr = bpr_loss(h_user, h_pos, h_neg)

        # Instance-level alignment
        loss_cmcl = infonce(h_mm[pos_items], h_id_i, temperature=0.1)

        # Distribution-level alignment
        cost = torch.cdist(h_mm[pos_items], h_id_i, p=2) ** 2
        P = sinkhorn(cost, epsilon=epsilon, n_iters=30)
        loss_oat = (P * cost).sum()

        loss = loss_bpr + 0.1 * loss_cmcl + 0.01 * loss_oat
        loss.backward()
        optimizer.step()
```

**Example 2: Content platform cold-start mitigation**

User: "We have a short-video platform where new videos have rich metadata (titles, tags, thumbnails) but zero interaction data. How can we recommend these cold-start items using our existing collaborative filtering model?"

Approach:
1. Encode video metadata with LLM -> `mm_emb` for all videos (new and existing)
2. For existing videos, also have trained ID embeddings from CF model
3. Train the RecGOAT alignment on existing videos: CMCL + OAT aligns `mm_emb` space to `H_id` space
4. For cold-start videos, project `mm_emb` through the learned alignment to synthesize pseudo-ID embeddings
5. Use pseudo-ID embeddings for ranking, with the graph attention encoder providing collaborative context from similar existing items

Output:
```python
# Cold-start inference
new_video_mm = sentence_model.encode(new_video_metadata)  # [1, 384]
new_video_mm = projection(new_video_mm)                    # [1, 256]

# Find k-nearest existing items in mm space
sims = cosine_similarity(new_video_mm, existing_mm_embs)
top_k_indices = sims.topk(20).indices

# Use graph attention to propagate collaborative signal from neighbors
# Add temporary edges from new video to top-k neighbors
pseudo_graph_emb = gat_encoder.single_node_forward(
    new_video_mm, neighbor_ids=top_k_indices, graph=existing_graph
)
# Fuse: the OAT-aligned space ensures mm and ID are distributionally compatible
pseudo_id = alignment_head(pseudo_graph_emb)  # Learned during OAT training
user_scores = (user_embeddings @ pseudo_id.T).squeeze()
```

**Example 3: Adding OAT alignment to an existing LightGCN recommender**

User: "I already have a LightGCN model trained. I want to add LLM features without retraining from scratch. Can I bolt on the alignment?"

Approach:
1. Freeze LightGCN embeddings as `H_id` (both user and item)
2. Generate LLM embeddings, project to same dimensionality
3. Train only the alignment components: a lightweight GAT (1 layer) + CMCL + OAT losses
4. At inference, add aligned multimodal embeddings to frozen LightGCN embeddings

Output:
```python
# Minimal bolt-on alignment module
class RecGOATAlignment(nn.Module):
    def __init__(self, mm_dim, id_dim, hidden_dim):
        super().__init__()
        self.mm_proj = nn.Linear(mm_dim, hidden_dim)
        self.gat = GATConv(hidden_dim, hidden_dim, heads=4, concat=False)
        self.align_head = nn.Linear(hidden_dim, id_dim)

    def forward(self, mm_features, edge_index):
        h = F.relu(self.mm_proj(mm_features))
        h = self.gat(h, edge_index)
        return self.align_head(h)  # Maps to ID embedding space

# Train with frozen H_id from LightGCN
aligned_mm = alignment_module(llm_embeddings, graph.edge_index)
loss = 0.1 * infonce(aligned_mm, frozen_h_id) + 0.01 * oat_loss(aligned_mm, frozen_h_id)
```

## Best Practices

- **Do**: Pre-compute and cache LLM embeddings offline. They are frozen during RecGOAT training, so regenerating them every epoch wastes compute. Store as memory-mapped numpy arrays for large catalogs.
- **Do**: Start with CMCL only, verify it improves over baseline, then add OAT. The contrastive loss is more stable; OAT can destabilize training if hyperparameters are wrong. This progressive approach matches the paper's dual-granularity philosophy.
- **Do**: Use the adaptive epsilon schedule for Sinkhorn. Start with `epsilon=0.1` (smooth, exploratory transport plans) and decay to `0.01` (sharp, precise alignment). Fixed epsilon often leads to either trivial transport plans or numerical instability.
- **Do**: Normalize embeddings before computing the OAT cost matrix. Without normalization, magnitude differences between LLM embeddings and ID embeddings dominate the transport cost, ignoring directional alignment.
- **Avoid**: Using OAT with very small batch sizes (< 256). Optimal transport estimates distribution-level properties; with too few samples, the transport plan is noisy and the gradient signal is unreliable.
- **Avoid**: Making the graph too dense. For item-item edges, set a co-interaction threshold (e.g., 5+ shared users). Dense graphs dilute attention weights and increase memory quadratically. Sparse, informative edges outperform dense, noisy ones.

## Error Handling

- **Sinkhorn divergence**: If OAT loss becomes NaN, the Sinkhorn iterations are numerically unstable. Increase `epsilon` (try 0.5), reduce Sinkhorn iterations to 10, or add `1e-8` to the cost matrix diagonal. Log-domain Sinkhorn is more stable for small epsilon values.
- **Contrastive collapse**: If CMCL loss drops to near-zero quickly, all embeddings are collapsing to a single point. Reduce learning rate, increase temperature `tau`, or add a uniformity regularizer (`-log(mean(exp(-2 * pdist(embeddings))))`.
- **Graph OOM**: Heterogeneous graphs with millions of nodes exhaust GPU memory. Use mini-batch graph sampling (GraphSAINT or NeighborLoader from PyG) rather than full-graph forward passes. Sample 2-hop neighborhoods of batch nodes.
- **Modality imbalance**: If text embeddings dominate over image embeddings (or vice versa), the alignment collapses to one modality. Add per-modality projection heads and a modality-balancing loss, or normalize each modality's contribution before fusion.
- **Cold-start regression**: If adding LLM features makes warm-item performance worse, the alignment is too aggressive. Reduce `lambda_1` and `lambda_2` by 10x, or use a gated fusion (`alpha * H_mm + (1-alpha) * H_id` with learned `alpha`) instead of additive fusion.

## Limitations

- Requires pre-computed LLM embeddings for all items. For catalogs with millions of items updated in real-time (e.g., news articles), the embedding generation pipeline becomes a bottleneck. Batch processing with a fast encoder (MiniLM, not GPT-4) is necessary.
- The Sinkhorn-based OAT scales as O(N^2) within each mini-batch for the cost matrix. For batch sizes above ~4096, consider approximations like sliced optimal transport or mini-batch OT.
- Graph construction assumes meaningful co-interaction patterns exist. For platforms with extremely sparse interaction data (< 5 interactions per user on average), the graph is too sparse for GAT to learn useful attention, and simpler concatenation may suffice.
- The theoretical guarantees in the paper assume the LLM embedding space and the ID embedding space are related by a smooth transport map. If the modalities are fundamentally disconnected (e.g., item IDs are randomly assigned with no semantic structure), OAT alignment adds no value.
- Training requires careful hyperparameter tuning of three loss weights, temperature, epsilon schedule, and graph construction thresholds. There is no single default configuration that works across domains.

## Reference

**Paper**: [RecGOAT: Graph Optimal Adaptive Transport for LLM-Enhanced Multimodal Recommendation with Dual Semantic Alignment](https://arxiv.org/abs/2602.00682v1) (Li et al., 2026). Focus on Section 3 (method) for the dual-granularity alignment formulation, Section 3.3 for the OAT derivation with Sinkhorn, and Section 4 for ablation studies showing the individual contribution of CMCL vs. OAT.

**Code**: https://github.com/6lyc/RecGOAT-LLM4Rec (pending release as of publication).