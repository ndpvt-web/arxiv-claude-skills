---
name: "empowering-contrastive-federated-sequential"
description: |
  Build federated sequential recommendation systems that use LLM-generated contrastive views
  (future trajectories, semantic rephrasings, counterfactual negatives) to improve next-item
  prediction without sharing raw user data. Based on the LUMOS architecture.

  Trigger phrases:
  - "Build a federated recommendation system with contrastive learning"
  - "Implement privacy-preserving sequential recommendation"
  - "Use LLMs to augment user interaction sequences for recommendation"
  - "Add contrastive views to a federated learning pipeline"
  - "Implement LUMOS-style tri-view contrastive optimization"
  - "Generate counterfactual negatives for recommendation training"
---

# Empowering Contrastive Federated Sequential Recommendation with LLMs (LUMOS)

This skill enables Claude to build federated sequential recommendation systems where each client device uses a local LLM to synthesize three complementary views of its interaction history — future-oriented trajectories, semantically equivalent rephrasings, and preference-inconsistent counterfactuals — then trains a shared recommendation backbone via tri-view contrastive optimization. The result is richer user representations without ever transmitting raw behavioral data or LLM-generated content to the server.

## When to Use

- When the user needs a next-item recommendation system that must operate under privacy constraints (GDPR, on-device data)
- When building a federated learning pipeline for sequential recommendation and struggling with sparse or noisy per-client interaction logs
- When the user wants to apply contrastive learning to recommendation but needs principled positive and negative pair construction
- When augmenting user behavior sequences with LLM-generated variants to improve cold-start or data-scarce clients
- When implementing a FedAvg-based recommendation system and looking for ways to improve model quality without server-side data processing
- When the user asks how to generate counterfactual negative sequences for contrastive training in recommendation

## Key Technique

**The core problem:** In federated sequential recommendation, each device holds only a single user's short, noisy interaction log. Standard FedAvg learns a shared model by averaging parameters, but per-client data scarcity severely limits representation quality. Manual augmentation (random item swaps, masking) adds noise without semantic grounding.

**LUMOS solution — parameter-isolated LLM augmentation:** Each client privately invokes an on-device LLM to produce three sequence variants from the original interaction history. (1) *Future-oriented trajectories* extrapolate plausible next interactions beyond the last recorded event, providing forward-looking supervision. (2) *Semantically equivalent rephrasings* substitute items with close alternatives or mildly reorder interactions, preserving user intent while diversifying patterns. (3) *Preference-inconsistent counterfactuals* generate sequences from opposite categories or unrelated domains, serving as hard negatives. All three variants stay on-device — only model weight updates are transmitted to the server.

**Tri-view contrastive optimization:** The four sequences (original + three variants) are encoded through the shared backbone (e.g., SASRec). An InfoNCE-style loss pulls the original representation toward the future and rephrased views (positives) and pushes it away from the counterfactual view (negative). The combined objective is `L = L_rec + λ_CL * L_CL`, where `L_rec` is the standard next-item prediction loss and `λ_CL` controls contrastive strength (default 0.1). This scheme enriches local training signals without increasing communication cost, and the semantically grounded negatives improve robustness under noisy and adversarial conditions.

## Step-by-Step Workflow

1. **Define the interaction schema.** Establish the item vocabulary (item IDs, categories, textual metadata) and the format for user interaction sequences (ordered lists of item IDs with timestamps). Set a maximum sequence length (default: 50 items) and minimum threshold (default: 5 items).

2. **Set up the federated backbone.** Implement a sequential recommendation model as the shared encoder — SASRec (self-attention) is the primary choice; GRU4Rec is a viable alternative. Each client holds a local copy of the model parameters. Use FedAvg for aggregation: sample 10% of clients per round, run 5 local epochs per client, then average parameters on the server.

3. **Implement the LLM sequence generator module.** Create a local module that takes a user's interaction history and produces three views via LLM prompting:

   - **Future-oriented prompt:** Given the sequence of items the user interacted with (with titles/categories), ask the LLM to predict 3–5 plausible next items and append them to the sequence.
   - **Semantic rephrasing prompt:** Ask the LLM to rewrite the sequence by substituting items with semantically similar alternatives (same category, similar attributes) or applying mild reordering, while preserving overall intent.
   - **Counterfactual negative prompt:** Ask the LLM to generate a sequence that this user would be unlikely to engage with — items from opposite categories, unrelated domains, or conflicting preference patterns.

4. **Map LLM text outputs back to item IDs.** Use fuzzy matching, embedding similarity, or a lookup index to convert the LLM's natural-language item descriptions back into valid item IDs from the vocabulary. Discard any items that cannot be mapped.

5. **Encode all four sequences.** Pass the original sequence and the three synthetic variants through the shared backbone encoder to obtain four representation vectors: `h_u` (original), `h_u_F` (future), `h_u_P` (rephrased), `h_u_N` (counterfactual).

6. **Compute the tri-view contrastive loss.** Use an InfoNCE-style formulation with cosine similarity and temperature τ = 0.07:
   ```
   L_CL = -log( exp(sim(h_u, h_u_F)/τ) / Z ) - log( exp(sim(h_u, h_u_P)/τ) / Z )
   ```
   where `Z = exp(sim(h_u, h_u_F)/τ) + exp(sim(h_u, h_u_P)/τ) + exp(sim(h_u, h_u_N)/τ)`.
   The counterfactual `h_u_N` participates only as a negative in the denominator.

7. **Combine losses and train locally.** Set the total loss as `L = L_rec + 0.1 * L_CL`. Use Adam optimizer (lr=1e-3, weight_decay=1e-5), batch size 128, gradient clipping at max norm 5.0. Train for 5 local epochs per communication round.

8. **Aggregate on the server.** Collect updated parameters from sampled clients and compute the unweighted average: `θ^(t+1) = (1/|U_t|) * Σ θ_u^(t)`. No synthetic data, prompts, or intermediate representations are transmitted — only model weights.

9. **Iterate for 100 communication rounds.** Evaluate using HR@K and NDCG@K (K=10, 20) on a held-out test set after each round to track convergence.

10. **Harden against noise.** The contrastive counterfactuals naturally provide adversarial robustness. For additional protection under poisoning attacks, monitor per-client loss distributions and clip outlier parameter updates during aggregation.

## Concrete Examples

**Example 1: E-commerce product recommendation**

User: "I'm building a federated recommendation system for a shopping app. Each user's purchase history stays on their phone. How do I improve recommendation quality without collecting data centrally?"

Approach:
1. Define item schema: `{id, title, category, brand}` with sequences of purchased item IDs.
2. Deploy SASRec (2 attention layers, 64-dim embeddings) as the federated backbone.
3. On each device, prompt the local LLM with the user's purchase history:
   - Future: "User bought [Nike Running Shoes, Adidas Shorts, Garmin Watch]. What 3 items would they likely buy next?" → mapped to item IDs for fitness accessories.
   - Rephrasing: "Rewrite this purchase sequence using similar but different products." → [Asics Running Shoes, Puma Shorts, Fitbit Watch].
   - Counterfactual: "Generate a purchase sequence this fitness-oriented user would NOT make." → [Oil Painting Set, Ceramic Vase, Knitting Kit].
4. Encode all four sequences, compute tri-view contrastive + next-item loss, train 5 epochs locally.
5. Upload only model weights to server for FedAvg aggregation.

Output: A federated model that achieves comparable or better HR@20 than a centralized SASRec, with no raw purchase data leaving the device.

**Example 2: News recommendation with MIND-style data**

User: "I have a news recommendation dataset split across users. I want to do federated training with contrastive augmentation."

Approach:
1. Schema: articles with `{id, title, category, subcategory}`, user reading sequences.
2. Use SASRec backbone. Minimum sequence length = 5 articles.
3. LLM augmentation per client:
   - Future: "User read [Tesla earnings article, EV market analysis, Battery tech review]. Predict next 3 articles." → mapped IDs for auto-industry articles.
   - Rephrasing: Substitute with articles from same subcategory: [Ford EV article, Clean energy analysis, Lithium mining review].
   - Counterfactual: Generate a sequence from a disinterested domain: [Celebrity gossip, Reality TV recap, Fashion week coverage].
4. Train with `L_rec + 0.1 * L_CL`, Adam lr=1e-3, τ=0.07.
5. FedAvg across 100 rounds, sampling 10% of clients per round.

Output: Improved NDCG@20 over baseline FedSeqRec, with natural robustness to click-bait noise in individual reading histories.

**Example 3: Adding LUMOS-style contrastive views to an existing codebase**

User: "I already have a FedAvg + SASRec pipeline in PyTorch. How do I add the tri-view contrastive component?"

Approach:
1. Add an `LLMAugmentor` class that takes a sequence of item IDs and returns three augmented sequences:
```python
class LLMAugmentor:
    def __init__(self, item_catalog, llm_client):
        self.catalog = item_catalog
        self.llm = llm_client

    def augment(self, seq: list[int]) -> dict:
        text_seq = [self.catalog[i]["title"] for i in seq]
        future = self._prompt_future(text_seq)
        rephrased = self._prompt_rephrase(text_seq)
        counterfactual = self._prompt_counterfactual(text_seq)
        return {
            "future": self._map_to_ids(future),
            "rephrased": self._map_to_ids(rephrased),
            "counterfactual": self._map_to_ids(counterfactual),
        }
```

2. Add the contrastive loss to the training loop:
```python
def tri_view_contrastive_loss(h_orig, h_future, h_rephrased, h_counter, tau=0.07):
    sim_f = F.cosine_similarity(h_orig, h_future) / tau
    sim_p = F.cosine_similarity(h_orig, h_rephrased) / tau
    sim_n = F.cosine_similarity(h_orig, h_counter) / tau
    denom = torch.exp(sim_f) + torch.exp(sim_p) + torch.exp(sim_n)
    loss = -torch.log(torch.exp(sim_f) / denom) - torch.log(torch.exp(sim_p) / denom)
    return loss.mean()
```

3. In the local training loop, call `augmentor.augment(seq)`, encode all four sequences through the SASRec encoder, compute `loss = rec_loss + 0.1 * cl_loss`, and backprop as usual. No changes needed to the FedAvg server code.

## Best Practices

- **Do:** Cache LLM-generated augmentations per sequence and regenerate only when the user's history changes significantly. LLM inference is expensive; avoid calling it every training epoch.
- **Do:** Validate that mapped item IDs from LLM outputs are valid vocabulary entries. Drop sequences where fewer than 50% of items can be mapped.
- **Do:** Use temperature τ = 0.07 for the contrastive loss and λ_CL = 0.1 as starting points, then tune on a validation set. Lower τ sharpens the similarity distribution; higher λ_CL gives more weight to contrastive learning vs. next-item prediction.
- **Do:** Enforce minimum sequence length (5 items). Clients with shorter histories produce unreliable augmentations and should fall back to standard training.
- **Avoid:** Sending LLM prompts, synthetic sequences, or intermediate representations to the server. The entire point of parameter isolation is that augmented data stays on-device.
- **Avoid:** Using random augmentation (item masking, random swaps) as a substitute for LLM-grounded generation. Random methods add noise without semantic coherence and underperform LLM-based views in LUMOS experiments.

## Error Handling

| Problem | Cause | Resolution |
|---------|-------|------------|
| LLM generates items not in vocabulary | Open-ended generation or domain drift | Use constrained decoding or a retrieval step to map outputs to nearest catalog items via embedding similarity |
| Contrastive loss dominates and degrades recommendation accuracy | λ_CL too high | Reduce λ_CL (try 0.01–0.05) or add a warmup period where only L_rec is active for the first 20 rounds |
| Counterfactual sequences are too similar to original | LLM fails to generate truly dissimilar content | Add explicit category exclusion in the prompt (e.g., "Do NOT include items from categories: [user's top 3 categories]") |
| Client training diverges after augmentation | Noisy or contradictory augmented sequences | Apply gradient clipping (max norm 5.0) and filter augmented sequences by a minimum semantic distance threshold from the original |
| FedAvg aggregation produces poor global model | High client heterogeneity | Consider FedProx (add proximal term to local loss) or increase local epochs to stabilize client updates before aggregation |

## Limitations

- **LLM dependency on-device:** The method assumes a capable LLM is available locally. On resource-constrained devices (phones, IoT), running even a small LLM may be impractical. Consider using a lightweight model (e.g., a quantized 1-3B parameter model) or pre-computing augmentations during charging/idle periods.
- **Item vocabulary mapping:** Converting free-text LLM outputs back to valid item IDs is a lossy process. Domains with highly structured or numeric item identifiers (e.g., SKUs) will need a robust retrieval or matching layer.
- **Cold-start clients:** Users with fewer than 5 interactions cannot produce meaningful augmentations. These clients should participate in FedAvg with standard training only.
- **Communication cost unchanged:** LUMOS improves model quality but does not reduce communication rounds or parameter transmission size. For bandwidth-constrained settings, combine with compression or sparse update techniques.
- **Evaluation scope:** The paper validates on Amazon product and MIND news datasets. Performance on domains with very different interaction patterns (e.g., music streaming, video) is not established.

## Reference

**Paper:** [Empowering Contrastive Federated Sequential Recommendation with LLMs](https://arxiv.org/abs/2602.09306v1) (Nguyen et al., 2026). Focus on Section 3 (LUMOS architecture), Section 3.3 (tri-view contrastive loss formulation), and Section 4 (experimental setup and hyperparameters) for implementation details.