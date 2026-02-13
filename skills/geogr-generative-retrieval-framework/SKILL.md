---
name: "geogr-generative-retrieval-framework"
description: "Build spatio-temporal POI recommendation systems using generative retrieval with geo-aware semantic IDs and LLM alignment. Use when: 'build a POI recommendation system', 'next location prediction with LLMs', 'geo-aware semantic tokenization for places', 'generative retrieval for location services', 'spatio-temporal collaborative filtering with language models', 'encode POIs as semantic IDs for autoregressive generation'."
---

# GeoGR: Generative Retrieval for Spatio-Temporal POI Recommendation

This skill enables Claude to design and implement generative retrieval systems for Point-of-Interest (POI) recommendation, following the GeoGR framework. The core idea: instead of scoring candidate POIs with a retrieval model, encode each POI as a short hierarchical semantic ID (SID) and train an LLM to autoregressively generate the next SID given a user's trajectory and context. This combines the expressiveness of language models with geographically-grounded item representations, achieving state-of-the-art next-POI prediction on sparse, real-world check-in datasets.

## When to Use

- When the user needs to build a next-POI or next-location recommendation system
- When the user wants to apply generative retrieval (predict item IDs token-by-token) to geospatial data
- When the user asks how to encode locations/places into semantic tokens for LLM consumption
- When the user is building navigation or map-based services that recommend destinations
- When the user wants to combine collaborative filtering signals with geographic proximity constraints
- When the user needs to align an LLM with a domain-specific token vocabulary (SIDs) for recommendation
- When the user asks about residual quantization (RQ-VAE/RQ-KMeans) for item tokenization

## Key Technique

**Stage 1 — Geo-Aware SID Tokenization.** Each POI is encoded into a compact 3-token semantic ID through a pipeline: (a) embed POI metadata (name, category, lat/lon, address) via an LLM encoder, (b) build co-visited POI pairs from user trajectories filtered by a geographic distance threshold (e.g., 3 km) and weighted by Swing similarity to suppress popular-item bias, (c) apply noise-contrastive loss over these pairs so geographically and behaviorally related POIs cluster together, and (d) quantize embeddings via 3-layer Residual K-Means (codebook sizes 32 / 64 / 4096) to produce a coarse-to-fine SID `[q1, q2, q3]`. An EM-style iterative refinement loop alternates between fine-tuning the LLM to predict current SIDs and updating SIDs via beam search to reduce collisions and improve downstream accuracy.

**Stage 2 — Multi-Stage LLM Training.** The LLM (e.g., Qwen-4B for quality, Qwen-0.6B for latency) is first aligned with SID tokens through Continued Pre-Training (CPT) on four template families: trajectory sequences ("At {time} visited <SID>; ..."), structured POI facts ("<SID>: Name: ...; Category: ..."), natural-language descriptions, and QA pairs. This teaches the LLM that SIDs are grounded entities with geographic meaning. Then Supervised Fine-Tuning (SFT) trains the model in an instruction-following format: given real-time context (GPS, timestamp, weather, search query) plus a short-term trajectory (last 32 POIs) and a summarized long-term profile, autoregressively generate the target SID's 3 tokens with a hierarchical negative log-likelihood loss.

**Why it works better.** Vanilla generative recommendation assigns random or category-only IDs, losing spatial structure. GeoGR's geo-constrained contrastive learning ensures nearby, co-visited POIs share SID prefixes, so the LLM's coarse-to-fine generation naturally narrows from region to neighborhood to specific venue. The EM refinement prevents SID collisions that plague one-shot quantization.

## Step-by-Step Workflow

1. **Prepare the POI dataset.** Collect POI metadata (ID, name, category, latitude, longitude, address) and user check-in sequences with timestamps. Split temporally: 80% train / 10% val / 10% test.

2. **Embed POI metadata.** For each POI, concatenate its attributes into a text string and encode it through a pre-trained LLM encoder (or sentence-transformer) to produce a dense embedding vector `e_pi`.

3. **Build geo-constrained co-visited pairs.** From user trajectories, extract pairs of POIs visited by the same user. Compute Swing similarity (downweights pairs involving universally popular items). Filter to retain only pairs within a geographic distance threshold (3 km works well). These are your positive pairs for contrastive learning.

4. **Train contrastive embeddings.** Apply InfoNCE loss over the positive pairs with in-batch negatives. Use temperature `τ` (tune between 0.05–0.2). Train until POI embeddings of geographically and behaviorally similar venues cluster tightly (verify with t-SNE).

5. **Quantize embeddings into 3-layer SIDs via Residual K-Means.** Run K-Means on the embedding space with K=32 for layer 1 (coarse region). Compute residuals, run K-Means with K=64 for layer 2 (neighborhood). Compute residuals again, run K-Means with K=4096 for layer 3 (specific venue). Each POI now has SID `[q1, q2, q3]`.

6. **Run EM-style SID refinement (2–5 iterations).** E-step: fine-tune a small LLM to predict SID from POI description text. M-step: for each POI, use beam search (beam width 20) to generate candidate SIDs; if the ground-truth SID is not among candidates, replace it with the highest-probability candidate. Track quantile accuracy and downstream Recall to detect convergence.

7. **Add SID tokens to the LLM vocabulary.** Extend the tokenizer with new special tokens for each codebook entry (e.g., `<SID_L1_0>` through `<SID_L1_31>`, etc.). Initialize their embeddings randomly or from the K-Means centroids.

8. **Run Continued Pre-Training (CPT).** Construct training samples from four templates: (a) trajectory sequences with SIDs and timestamps, (b) structured POI attribute cards, (c) natural-language POI descriptions, (d) QA pairs asking "What is <SID>?". Train with standard autoregressive language modeling loss across all templates.

9. **Run Supervised Fine-Tuning (SFT).** Format each training example as instruction / input / response: the instruction states the next-POI prediction task, the input contains real-time context (location, time, weather, query) plus the user's recent 32-POI trajectory and a summarized long-term profile, and the response is the target SID's 3 tokens. Train with hierarchical NLL: `L = -Σ_{i=1}^{3} log p(q_i | prompt, q_{<i})`.

10. **Serve recommendations via beam search.** At inference, construct the prompt from live user context, generate top-50 SIDs with dynamic beam search, map SIDs back to POI IDs, and pass candidates to a downstream re-ranker for final ordering.

## Concrete Examples

**Example 1: Building a POI tokenizer from Foursquare check-in data**

User: "I have a Foursquare NYC dataset with 5K POIs and 100K check-ins. How do I create semantic IDs for each POI?"

Approach:
1. Load POI metadata and encode each POI's `(name, category, lat, lon, address)` through a sentence-transformer (e.g., `all-MiniLM-L6-v2`) to get 384-dim embeddings.
2. Extract co-visited POI pairs from user trajectories. Compute Swing similarity. Filter to pairs within 3 km.
3. Fine-tune embeddings with InfoNCE contrastive loss (batch size 512, τ=0.1, 20 epochs).
4. Apply 3-layer Residual K-Means (32 / 64 / 4096) to produce SIDs.
5. Run 3 rounds of EM refinement using a small LM (e.g., GPT-2 124M) to reduce collisions.

Output:
```python
# poi_sids.json (excerpt)
{
  "poi_4231": [12, 41, 3802],   # Times Square area, restaurant cluster
  "poi_4232": [12, 41, 1567],   # Nearby restaurant, shares L1+L2 prefix
  "poi_891":  [3, 17, 2201],    # Brooklyn venue, different L1 prefix
}
# POIs in the same neighborhood share the first 1-2 SID tokens.
```

**Example 2: Fine-tuning Qwen for next-POI generation**

User: "I have SIDs for my POIs. How do I train an LLM to predict the next POI?"

Approach:
1. Extend the Qwen-0.5B tokenizer with SID special tokens (32 + 64 + 4096 = 4192 new tokens).
2. Build CPT dataset with four template types from training trajectories and POI metadata.
3. Run CPT for 2 epochs (LR 2e-5, cosine schedule) to ground SID tokens.
4. Build SFT dataset: each sample is `(instruction, context + trajectory, target SID)`.
5. Run SFT for 3 epochs (LR 1e-5) with hierarchical NLL loss over the 3 SID tokens.

Output:
```
# Prompt at inference time:
"Task: Predict the next POI the user will visit.
Context: Location: (40.748, -73.986), Time: 2024-03-15 12:30, Weather: Sunny
Recent visits: <SID_5_22_1801> at 09:00; <SID_5_22_3455> at 10:15; <SID_12_41_982> at 11:30
Profile: Frequent diner, prefers Asian cuisine, visits Midtown 3x/week.
Next POI:"

# Model generates: <SID_12_41_3802>  → maps to "Sushi Nakazawa, West Village"
```

**Example 3: Implementing the contrastive learning step**

User: "Show me how to implement the geo-constrained contrastive loss for POI pairs."

```python
import torch
import torch.nn.functional as F

def geo_constrained_infonce(embeddings, positive_pairs, temperature=0.1):
    """
    embeddings: (N, D) tensor of POI embeddings
    positive_pairs: list of (i, j) index pairs within geo threshold
    """
    losses = []
    for i, j in positive_pairs:
        e_i = embeddings[i]  # anchor
        e_j = embeddings[j]  # positive
        # All other POIs in batch as negatives
        neg_mask = torch.ones(len(embeddings), dtype=torch.bool)
        neg_mask[i] = False
        neg_mask[j] = False
        negatives = embeddings[neg_mask]

        pos_sim = torch.dot(e_i, e_j) / temperature
        neg_sims = torch.mv(negatives, e_i) / temperature
        logits = torch.cat([pos_sim.unsqueeze(0), neg_sims])
        labels = torch.zeros(1, dtype=torch.long, device=logits.device)
        losses.append(F.cross_entropy(logits.unsqueeze(0), labels))

    return torch.stack(losses).mean()


def filter_pairs_by_distance(pairs, poi_coords, max_km=3.0):
    """Retain only co-visited pairs within max_km of each other."""
    from math import radians, sin, cos, sqrt, atan2
    filtered = []
    for i, j in pairs:
        lat1, lon1 = map(radians, poi_coords[i])
        lat2, lon2 = map(radians, poi_coords[j])
        dlat, dlon = lat2 - lat1, lon2 - lon1
        a = sin(dlat/2)**2 + cos(lat1)*cos(lat2)*sin(dlon/2)**2
        km = 6371 * 2 * atan2(sqrt(a), sqrt(1-a))
        if km <= max_km:
            filtered.append((i, j))
    return filtered
```

## Best Practices

- **Do:** Use Swing similarity (not raw co-occurrence counts) when building POI pairs. It downweights universally popular venues that add noise.
- **Do:** Keep the geographic distance threshold tight (1–5 km). Wider thresholds dilute the spatial signal and produce SIDs where distant POIs share prefixes.
- **Do:** Run at least 2–3 EM refinement iterations. The first quantization pass has high collision rates; refinement cuts them significantly.
- **Do:** Use all four CPT template types. Ablation shows removing any single template degrades performance — each teaches the LLM a different facet of SID semantics.
- **Avoid:** Skipping the CPT stage and jumping straight to SFT. The LLM treats SID tokens as random noise without CPT grounding, and SFT alone cannot compensate.
- **Avoid:** Using a single flat codebook instead of hierarchical residual quantization. A flat ID space forces the LLM to predict among thousands of tokens in one step; the 3-layer hierarchy (32 → 64 → 4096) makes generation tractable via coarse-to-fine narrowing.

## Error Handling

- **SID collisions (multiple POIs share the same SID):** This is the most common failure mode. Detect by counting unique SIDs vs. total POIs after quantization. If collision rate exceeds 5%, increase the layer-3 codebook size or run additional EM refinement rounds.
- **LLM generates invalid SID tokens:** At inference, the model may produce token combinations that don't map to any POI. Constrain decoding with a prefix tree (trie) built from all valid SIDs so every generated sequence resolves to a real POI.
- **Sparse trajectory data:** Users with fewer than 5 check-ins yield poor training signals. Augment short trajectories by injecting popular POIs from the same geohash region, or exclude ultra-sparse users from training and handle them with a popularity-based fallback.
- **Embedding collapse during contrastive learning:** If all embeddings converge to a single point, reduce learning rate, increase temperature τ, or add more hard negatives. Monitor embedding variance each epoch.
- **Latency at serving time:** Full-size LLMs (4B+ params) may exceed latency budgets. Distill to a smaller model (0.5B–1B) after SFT. The paper shows Qwen-0.6B achieves ~50ms per request with minimal accuracy loss.

## Limitations

- **Requires check-in trajectory data.** The framework cannot cold-start without user visit sequences. Pure metadata-based approaches are needed for new platforms with no behavioral data.
- **Geographic bias in SID structure.** SIDs encode spatial proximity, so POIs in under-represented regions get poor embeddings. This matters for global deployments with uneven data density.
- **LLM alignment cost.** CPT + SFT requires substantial compute. For datasets under 10K POIs, simpler sequential models (SASRec, GRU4Rec) may be more cost-effective.
- **Static SIDs.** Once assigned, SIDs don't update automatically when new POIs are added. New venues require re-running quantization or maintaining a separate index for unseen items.
- **Domain-specific.** The technique is designed for geographic POI recommendation. Applying it to non-spatial domains (e.g., product recommendation) loses the geographic constraint advantage that makes it work.

## Reference

**Paper:** [GeoGR: A Generative Retrieval Framework for Spatio-Temporal Aware POI Recommendation](https://arxiv.org/abs/2602.10411v1) — Wang et al., 2026. Focus on Section 3 (methodology) for the SID tokenization pipeline and Algorithm 1 for the EM refinement loop, and Section 4.4 for ablation studies showing each component's contribution.