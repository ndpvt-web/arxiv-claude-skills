---
name: "lmmrec-llm-driven-motivation-aware-multimodal"
description: "Build motivation-aware recommendation systems that use LLM chain-of-thought prompting to extract user and item motivations from reviews, then align textual and behavioral signals via contrastive learning. Use when: 'build a recommendation system from user reviews', 'extract purchase motivations with LLM', 'align review text with interaction data for recommendations', 'improve collaborative filtering with LLM-derived features', 'multimodal recommendation with text and behavior signals', 'use chain-of-thought to understand why users buy items'."
---

# LMMRec: LLM-Driven Motivation-Aware Multimodal Recommendation

This skill enables Claude to design and implement recommendation systems that go beyond surface-level collaborative filtering by extracting *why* users interact with items -- their underlying motivations -- using LLM chain-of-thought prompting on review text. It then aligns these textual motivation signals with interaction-graph signals through a dual-encoder architecture with contrastive learning and momentum-stabilized correspondence, following the LMMRec framework from Di et al. (2026).

## When to Use

- When a user wants to build a recommendation system that leverages user reviews or textual feedback, not just click/purchase logs
- When the user asks to extract purchase motivations, user intent, or preference explanations from review text using an LLM
- When building a hybrid recommender that combines text-derived features with collaborative filtering signals
- When the user wants to improve an existing graph-based recommender (e.g., LightGCN) by incorporating semantic understanding from reviews
- When asked to generate explainable recommendations that can articulate *why* an item is recommended
- When dealing with noisy multimodal data (e.g., inconsistent reviews vs. behavior) and needing robust cross-modal alignment

## Key Technique

**Motivation extraction via chain-of-thought prompting.** LMMRec uses an LLM (e.g., GPT-4o-mini) to read user reviews for each item and extract structured motivations -- what specifically attracted or repelled the user. The prompt guides the LLM through chain-of-thought reasoning to produce two outputs per review: positive interests (aspects the user liked) and negative interests (aspects the user disliked). These per-review motivations are then aggregated across all items a user interacted with to form a holistic user motivation profile. Item motivations are extracted similarly from the item's received reviews and descriptions. The text embeddings are generated using a model like `text-embeddings-3-large`.

**Dual-encoder cross-modal alignment.** The framework maintains two parallel encoders: a *text encoder* (MLP with LeakyReLU) that maps LLM-derived motivation embeddings into a shared space, and a *graph encoder* (LightGCN) that captures behavioral motivations from user-item interaction graphs. Behavioral motivations are modeled through prototype mapping -- a set of learnable motivation prototypes that the graph embeddings are softmax-projected onto, yielding a mixture-of-motivations representation per user/item.

**Noise-robust alignment with MCS and ITCM.** Two mechanisms keep the modalities aligned despite noise. The Motivation Coordination Strategy (MCS) uses InfoNCE-style contrastive loss to pull together the text and graph representations of the same user/item while pushing apart unrelated pairs. The Interaction-Text Correspondence Method (ITCM) adds a momentum-updated teacher model that produces stable soft targets, and the student model minimizes KL divergence against these targets bidirectionally -- preventing the semantic drift that occurs when both encoders update simultaneously on noisy data.

## Step-by-Step Workflow

1. **Collect and structure interaction data.** Organize user-item interactions into an adjacency matrix and gather all associated review text. Filter sparse users/items (e.g., require minimum 2-3 interactions). Split data 3:1:1 into train/validation/test sets.

2. **Design chain-of-thought prompts for motivation extraction.** Write prompts that instruct the LLM to read a user's review of an item and output: (a) a list of positive motivations (features/qualities the user valued), and (b) a list of negative motivations (features/qualities the user disliked or found lacking). The prompt should force structured output (JSON) and encourage reasoning before conclusions.

   ```
   System: You are an analyst extracting purchase motivations from product reviews.

   User: Analyze this review and extract the user's motivations.
   Review: "{review_text}"
   Item: "{item_name}" in category "{category}"

   Think step by step:
   1. What specific features or qualities did the reviewer value?
   2. What specific features or qualities disappointed the reviewer?
   3. What underlying need or preference drove this interaction?

   Output JSON:
   {
     "positive_motivations": ["..."],
     "negative_motivations": ["..."],
     "underlying_need": "..."
   }
   ```

3. **Aggregate motivations into user and item profiles.** For each user, collect all per-review motivations across their interaction history and summarize them into a unified motivation profile using a second LLM call (the "Amass" step). For items, aggregate motivations from all received reviews. Embed these text profiles using a sentence embedding model (e.g., `text-embeddings-3-large`) to produce dense vectors.

4. **Build the text encoder.** Implement a 2-layer MLP with LeakyReLU activation that maps the high-dimensional text embeddings into the shared motivation space (e.g., 32 dimensions):
   ```python
   class TextEncoder(nn.Module):
       def __init__(self, input_dim, hidden_dim, output_dim):
           super().__init__()
           self.fc1 = nn.Linear(input_dim, hidden_dim)
           self.fc2 = nn.Linear(hidden_dim, output_dim)
           self.act = nn.LeakyReLU()

       def forward(self, x):
           return self.fc2(self.act(self.fc1(x)))
   ```

5. **Build the graph encoder with motivation prototypes.** Implement LightGCN for message passing over the user-item bipartite graph. After obtaining node embeddings from K layers of propagation, project them onto I learnable motivation prototypes via softmax attention:
   ```python
   # prototypes: (num_prototypes, embed_dim)
   # node_embed: (batch, embed_dim)
   attn = torch.softmax(node_embed @ prototypes.T, dim=-1)  # (batch, num_prototypes)
   motivation_embed = attn @ prototypes  # (batch, embed_dim)
   ```

6. **Implement the Motivation Coordination Strategy (MCS).** Add an InfoNCE contrastive loss that aligns text-encoder and graph-encoder outputs for the same entity (user or item) while separating different entities. Compute three contrastive terms -- one for users, one for positively-interacted items, one for negatively-interacted items -- and sum them:
   ```python
   def mcs_loss(text_embeds, graph_embeds, temperature=0.2):
       sim = torch.mm(text_embeds, graph_embeds.T) / temperature
       labels = torch.arange(sim.size(0), device=sim.device)
       return F.cross_entropy(sim, labels)
   ```

7. **Implement the Interaction-Text Correspondence Method (ITCM).** Create momentum-updated copies of both encoders (the "teacher"). On each forward pass, the teacher produces soft probability distributions over correspondence; the student minimizes KL divergence against these targets. Update teacher weights with exponential moving average (e.g., momentum 0.995):
   ```python
   # Teacher EMA update
   for t_param, s_param in zip(teacher.parameters(), student.parameters()):
       t_param.data = 0.995 * t_param.data + 0.005 * s_param.data

   # Bidirectional KL loss
   student_logits_ab = student_text(x) @ student_graph(y).T
   teacher_logits_ab = teacher_text(x) @ teacher_graph(y).T
   loss_itcm = (
       F.kl_div(F.log_softmax(student_logits_ab, dim=-1),
                 F.softmax(teacher_logits_ab.detach(), dim=-1), reduction='batchmean')
       + F.kl_div(F.log_softmax(student_logits_ab.T, dim=-1),
                   F.softmax(teacher_logits_ab.T.detach(), dim=-1), reduction='batchmean')
   )
   ```

8. **Combine losses and train end-to-end.** The total loss is: `L = L_mcs + gamma * L_itcm + lambda * ||params||^2`. Use Adam optimizer (lr=0.003), batch size 512, embedding dim 32, and early stopping on validation Recall@10.

9. **Generate recommendations at inference.** Compute final user/item representations by summing or concatenating the graph-encoder and text-encoder outputs. Rank items by inner product with the user vector. Optionally surface the motivation prototypes as explanation text.

10. **Evaluate with ranking metrics.** Measure Recall@K and NDCG@K (K=5,10,20) on the held-out test set. Test robustness by injecting 5-30% random noise into the interaction matrix and verifying performance degrades gracefully.

## Concrete Examples

**Example 1: Restaurant recommendation from Yelp reviews**

User: "I have a Yelp dataset with user reviews and ratings. Build me a recommendation system that understands *why* people like certain restaurants."

Approach:
1. Filter users with >= 2 reviews. Build user-restaurant interaction graph.
2. For each review, prompt the LLM:
   ```
   Review: "Great craft beer selection but the wait was awful. Loved the outdoor patio."
   -> positive_motivations: ["craft beer variety", "outdoor dining atmosphere"]
   -> negative_motivations: ["long wait times"]
   -> underlying_need: "casual social dining with good drinks"
   ```
3. Aggregate per-user: User_42 motivation profile = "Values craft beverages, outdoor ambiance, dislikes slow service across 12 restaurants."
4. Embed profiles, train dual encoders with MCS+ITCM alignment against LightGCN interaction encoder.
5. At inference, recommend restaurants whose motivation prototypes (e.g., "ambiance-focused", "drink-variety") align with User_42's motivation vector.

Output:
```
Top-3 for User_42:
1. The Hopyard (score: 0.92) -- matches: craft beer, patio dining
2. Sunset Brewery (score: 0.87) -- matches: beer variety, outdoor seating
3. Garden Bistro (score: 0.83) -- matches: ambiance, quick service
```

**Example 2: Adding LMMRec to an existing book recommender**

User: "I already have a LightGCN model for Amazon book recommendations. How do I add motivation-awareness from review text?"

Approach:
1. Keep the existing LightGCN as the graph encoder -- no retraining needed initially.
2. Extract motivations from book reviews via LLM:
   ```
   Review: "Compelling world-building but the pacing dragged in the middle third."
   -> positive_motivations: ["immersive world-building", "rich fantasy setting"]
   -> negative_motivations: ["slow pacing in middle section"]
   ```
3. Aggregate into user/book motivation profiles and embed with `text-embeddings-3-large`.
4. Add a TextEncoder MLP that maps embeddings to the same dimension as LightGCN output (32-d).
5. Add MCS contrastive loss between text and graph embeddings during fine-tuning.
6. Add ITCM with momentum teacher copies of both encoders to prevent drift.
7. Joint fine-tune with combined loss. The graph encoder weights update slowly due to ITCM regularization.

Output improvement (typical):
```
Before (LightGCN only):  Recall@10 = 0.0812, NDCG@10 = 0.0534
After  (+ LMMRec):       Recall@10 = 0.0847, NDCG@10 = 0.0559  (+4.3%, +4.7%)
```

**Example 3: Motivation extraction prompt engineering**

User: "Help me write the chain-of-thought prompt for extracting gaming motivations from Steam reviews."

Output:
```python
MOTIVATION_EXTRACTION_PROMPT = """
You are analyzing a Steam game review to understand the player's motivations.

Game: {game_title}
Tags: {game_tags}
Review: "{review_text}"
Playtime: {hours} hours | Recommended: {recommended}

Reason step by step:
1. What gameplay elements does the reviewer specifically praise or criticize?
2. What emotional experience were they seeking (challenge, relaxation, story, social)?
3. Does their playtime vs. recommendation reveal hidden motivations?

Return JSON:
{{
  "positive_motivations": ["specific motivation 1", "specific motivation 2"],
  "negative_motivations": ["specific motivation 1"],
  "primary_need": "one phrase describing core motivation",
  "confidence": 0.0-1.0
}}
"""
```

## Best Practices

- **Do:** Use structured JSON output from the LLM for motivation extraction -- this ensures consistent parsing and aggregation across thousands of reviews.
- **Do:** Start with a small number of motivation prototypes (I=4-8) and increase only if validation metrics improve. Too many prototypes fragment the motivation space.
- **Do:** Apply the momentum EMA update (ITCM) with a high coefficient (0.99-0.999) for stability. The teacher model should change slowly.
- **Do:** Normalize embeddings before computing contrastive loss to prevent magnitude-driven shortcuts.
- **Avoid:** Running LLM extraction on every training epoch. Extract motivations once as a preprocessing step, embed them, and cache the vectors.
- **Avoid:** Using the same temperature parameter for MCS and ITCM losses. Tune them independently -- MCS typically needs lower temperature (0.1-0.2) while ITCM works better with moderate values (0.5-1.0).
- **Avoid:** Skipping the aggregation ("Amass") step. Individual review motivations are noisy; the summarization step across a user's full history is critical for stable profiles.

## Error Handling

- **LLM returns malformed JSON for motivation extraction:** Wrap extraction calls with retry logic and JSON validation. Fall back to a simpler prompt without chain-of-thought if the model consistently fails for a particular review. Set a `confidence` threshold and discard low-confidence extractions.
- **Contrastive loss collapses (all embeddings become similar):** This happens when temperature is too low or batch size too small. Increase temperature to 0.5 and ensure batch size >= 256. Add gradient clipping at 1.0.
- **Text and graph encoders diverge instead of aligning:** The ITCM momentum coefficient is too low -- increase it toward 0.999. Also verify that both encoders output embeddings in the same dimensionality and normalization scheme.
- **Sparse users have unreliable motivation profiles:** For users with only 1-2 reviews, fall back to item-side motivations (aggregate motivations from the items they interacted with) rather than relying on thin user-side text.
- **Embedding dimension mismatch:** The text embedding model (e.g., 3072-d from `text-embeddings-3-large`) must be projected down to match the graph encoder dimension (e.g., 32-d). The TextEncoder MLP handles this, but verify shapes before training.

## Limitations

- **Requires review text.** The core value of LMMRec comes from extracting motivations from textual feedback. Datasets with only implicit feedback (clicks, views) without any text cannot benefit from the motivation extraction pipeline. You would need to generate synthetic text from metadata as a workaround.
- **LLM extraction cost.** Processing reviews through GPT-4o-mini or similar models at scale (millions of reviews) incurs meaningful API cost. This is a one-time preprocessing step, but budget accordingly -- batch API endpoints with 50% discounts help.
- **Cold-start for new users/items.** Users or items with zero reviews have no text signal. The framework degrades to pure collaborative filtering in these cases.
- **Motivation prototypes are dataset-specific.** Prototypes learned on Yelp restaurant data do not transfer to Amazon books. Retrain prototypes per domain.
- **Marginal gains on already-strong baselines.** The 2-5% improvement is meaningful at scale but may not justify the added complexity for small-scale applications or prototypes.

## Reference

**Paper:** Di et al., "LMMRec: LLM-driven Motivation-aware Multimodal Recommendation" (2026). [arXiv:2602.05474v2](https://arxiv.org/abs/2602.05474v2)

Look for: Section 3 for the full framework architecture, Section 3.2 for the chain-of-thought motivation extraction pipeline, Section 3.3-3.4 for MCS contrastive loss and ITCM momentum correspondence formulations, and Section 4 for ablation studies showing the contribution of each component.