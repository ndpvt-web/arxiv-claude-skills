---
name: "polarmem-training-free-polarized-latent"
description: "Build polarized memory systems for multimodal agents that encode both positive and negative evidence as graph constraints, suppressing hallucinations without retraining. Use when: 'add negative evidence to RAG memory', 'build verifiable retrieval for VLM agent', 'suppress hallucinations in multimodal retrieval', 'polarized graph memory for agent', 'encode negation constraints in memory', 'logic-dominant retrieval pipeline'."
---

# PolarMem: Training-Free Polarized Latent Graph Memory

This skill teaches you to implement PolarMem-style memory systems that transform noisy VLM confidence scores into discrete logical states (verified-positive, verified-negative, uncertain) and store them in a polarized graph where inhibitory edges explicitly encode what is *not* true. At retrieval time, negative constraints dominate semantic similarity, categorically suppressing hallucination-prone results regardless of how "close" they appear in embedding space. The technique is training-free and wraps around any frozen vision-language model.

## When to Use

- When building a multimodal RAG system and the agent hallucinates by retrieving semantically similar but factually wrong evidence (e.g., retrieving a photo of a robin when asked about a cardinal because both are "red birds")
- When the user asks to add negation or "verified absence" tracking to a retrieval-augmented agent
- When implementing a memory layer for a long-horizon agent that must distinguish between "I saw X" and "I confirmed X is not here"
- When building a knowledge graph where edges must represent both presence and absence relationships
- When the user wants to reduce false positives in image retrieval by encoding what each image does *not* contain
- When creating a verifiable agent whose answers can be traced back to logically grounded evidence, not just cosine similarity

## Key Technique

**The core problem:** Standard dense retrieval (cosine similarity over embeddings) conflates *semantic affinity* with *factual existence*. An image of a forest may score highly for the query "tiger in forest" simply because forests are semantically close to tiger habitats -- even if no tiger is present. Probabilistic VLMs compound this by returning soft scores that never firmly say "no."

**PolarMem's solution has three stages.** First, *non-parametric distributional partitioning* runs ensemble VQA queries against each stored image (e.g., "Does this image contain a tiger?" across multiple prompt templates), collects confidence scores, then applies Otsu's method (maximum inter-class variance thresholding) to split the score distribution into three bins: verified-positive (above threshold + margin), verified-negative (below threshold - margin), and uncertain. The dynamic margin `delta = kappa * sigma_weighted` adapts to each image's score spread. This converts fuzzy likelihoods into discrete logical states without any learned parameters.

**Second,** these states are stored in a *polarized graph topology*. Visual episode nodes connect to concept nodes via excitatory edges (`HAS`) for verified-positive concepts and inhibitory edges (`NOT_HAS`) for verified-negative ones. The graph is heterogeneous: visual nodes, textual nodes, and concept nodes each carry typed embeddings. The key insight is that `NOT_HAS` edges are first-class citizens, not the absence of a `HAS` edge. **Third,** at retrieval time, a *lexicographic ranking protocol* sorts candidates by `(logical_state, semantic_similarity)` where logical state takes strict priority: any candidate whose verified-positive concepts intersect with the query's forbidden concepts, or whose verified-negative concepts intersect with the query's required concepts, receives `s_log = -1` and is categorically suppressed, regardless of embedding similarity.

## Step-by-Step Workflow

1. **Define your concept vocabulary.** For your domain, enumerate the set of concepts that matter (object classes, attributes, scene types). Keep this focused -- 50-200 concepts is practical. Each concept becomes a potential node in the graph.

2. **Run ensemble verification queries.** For each stored item (image, document chunk, etc.), query your VLM with multiple prompt templates per concept: `"Does this image contain {concept}? Answer Yes or No."` Use 3-5 template variations and average the `P("Yes")` scores to get a consistency score `sc` per concept per item.

3. **Apply Otsu's thresholding with dynamic margin.** Collect all `sc` values for a given item into a distribution. Inject anchor priors `{0.0, 1.0}` to stabilize sparse distributions. Find threshold `tau*` that maximizes inter-class variance. Compute margin `delta = kappa * sigma_weighted` (kappa=1.0 is a good default). Classify each concept:
   - `sc > tau* + delta` --> verified-positive (the item HAS this concept)
   - `sc < tau* - delta` --> verified-negative (the item does NOT HAVE this concept)
   - otherwise --> uncertain (do not store an edge)

4. **Construct the polarized graph.** Create nodes for each stored item and each concept. Add `HAS` edges from items to their verified-positive concepts. Add `NOT_HAS` edges from items to their verified-negative concepts. Store the item's embedding (from VLM encoder) on the node for later similarity computation.

5. **Build the semantic index.** Insert item embeddings into a vector store (Milvus, FAISS, pgvector) for fast approximate nearest-neighbor lookup. This handles the `s_sem` component of retrieval.

6. **Parse queries into required and forbidden concepts.** At query time, extract `Q+` (concepts the user wants present) and `Q-` (concepts that should be absent). For example, "Show me images of dogs but not cats" yields `Q+ = {dog}`, `Q- = {cat}`.

7. **Retrieve candidates and compute logical state.** Pull top-K candidates by embedding similarity. For each candidate, compute `s_log`:
   - If any concept in `Q+` appears in the candidate's `NOT_HAS` edges: `s_log = -1` (the memory says this concept is confirmed absent)
   - If any concept in `Q-` appears in the candidate's `HAS` edges: `s_log = -1` (the memory says this forbidden concept is confirmed present)
   - If any concept in `Q+` appears in the candidate's `HAS` edges: `s_log = +1` (verified match)
   - Otherwise: `s_log = 0` (no constraint applies)

8. **Lexicographic sort and filter.** Sort candidates by `(s_log descending, s_sem descending)`. All `s_log = -1` candidates are suppressed. Among the rest, `s_log = +1` candidates rank above `s_log = 0` regardless of semantic score.

9. **Serialize context and generate.** Take the top-K surviving candidates, serialize their content (image references, text chunks, verified concept lists) into the VLM prompt, and generate the final answer. The context is now logically sanitized.

10. **Optionally expose verification traces.** For each retrieved item in the context, include its `HAS` and `NOT_HAS` concept lists so downstream reasoning or users can audit *why* an item was included or excluded.

## Concrete Examples

**Example 1: Biological image retrieval with negation**

User: "I have a corpus of 10,000 wildlife images indexed with CLIP embeddings. When I ask 'show me images of eagles,' I keep getting hawks and falcons because they're semantically similar. How do I fix this?"

Approach:
1. Define concept vocabulary: `{eagle, hawk, falcon, owl, sparrow, ...}` (bird species relevant to corpus).
2. For each image, run ensemble VQA: query Qwen2.5-VL with 4 templates like `"Is there an eagle in this image?"`, `"Does this image show an eagle?"`, etc. Average P("Yes") scores.
3. Apply Otsu's partitioning per image. An image of a hawk will score `sc(eagle) = 0.12` (below `tau* - delta`), yielding a `NOT_HAS(eagle)` edge, and `sc(hawk) = 0.91` (above `tau* + delta`), yielding a `HAS(hawk)` edge.
4. At query time, parse "show me images of eagles" as `Q+ = {eagle}`.
5. Hawk images have `NOT_HAS(eagle)`, so `s_log = -1`. They are suppressed even though CLIP puts them at cosine similarity 0.92 with "eagle."
6. Only images with `HAS(eagle)` or no constraint (`s_log >= 0`) surface.

Output:
```
Retrieved 12 images (from 10,000 corpus):
  - img_4821.jpg  [s_log=+1, s_sem=0.89]  HAS: {eagle, tree, sky}
  - img_7103.jpg  [s_log=+1, s_sem=0.86]  HAS: {eagle, mountain}
  - img_0392.jpg  [s_log= 0, s_sem=0.84]  (no eagle constraint either way)
  ...
Suppressed 43 candidates with NOT_HAS(eagle): hawks (31), falcons (9), kites (3)
```

**Example 2: Document QA with verified negation**

User: "Build a RAG pipeline for medical records where the agent must never claim a patient has a condition that was explicitly ruled out."

Approach:
1. Concept vocabulary: ICD-10 condition codes relevant to the patient population.
2. For each document chunk, run ensemble queries: `"Does this record mention the patient HAS {condition}?"` across templates.
3. Partition: A radiology report saying "no evidence of pneumothorax" will produce `sc(pneumothorax) ≈ 0.08` --> `NOT_HAS(pneumothorax)`. A report confirming "bilateral pleural effusion" produces `sc(pleural_effusion) ≈ 0.94` --> `HAS(pleural_effusion)`.
4. When the agent processes the query "Does the patient have pneumothorax?", `Q+ = {pneumothorax}`.
5. Chunks with `NOT_HAS(pneumothorax)` get `s_log = -1` and are suppressed from the context window, preventing the VLM from seeing text about pneumothorax in a negation context and misinterpreting it as positive evidence.
6. The agent retrieves only chunks that either confirm pneumothorax or are neutral, then generates an answer grounded in that filtered evidence.

Output:
```
Query: "Does the patient have pneumothorax?"
Retrieved context (3 chunks):
  - record_42.txt:L45-60  [s_log=0, s_sem=0.71]  "Chest X-ray findings..."
  - record_42.txt:L120-135 [s_log=0, s_sem=0.65]  "Follow-up imaging..."
Suppressed: record_42.txt:L80-95 [NOT_HAS(pneumothorax)] "...no pneumothorax seen..."
Answer: "Based on available records, there is no confirmed diagnosis of pneumothorax."
```

**Example 3: Implementing the Otsu partitioning in Python**

User: "Show me how to implement the distributional partitioning step."

```python
import numpy as np

def polarized_partition(scores: dict[str, float], kappa: float = 1.0):
    """
    Partition concept confidence scores into positive, negative, uncertain.

    Args:
        scores: {concept_name: avg_vlm_confidence} for one item
        kappa: margin multiplier (higher = wider uncertain band)

    Returns:
        (verified_positive, verified_negative, uncertain) concept sets
    """
    if not scores:
        return set(), set(), set()

    values = np.array(list(scores.values()))
    # Anchor regularization: inject priors to stabilize sparse distributions
    anchored = np.concatenate([values, [0.0, 1.0]])

    # Otsu's method: find threshold maximizing inter-class variance
    best_tau, best_var = 0.5, -1.0
    for tau in np.linspace(0.01, 0.99, 200):
        w0 = anchored[anchored <= tau]
        w1 = anchored[anchored > tau]
        if len(w0) == 0 or len(w1) == 0:
            continue
        var_between = len(w0) * len(w1) * (w0.mean() - w1.mean()) ** 2
        if var_between > best_var:
            best_var = var_between
            best_tau = tau

    # Dynamic margin from weighted standard deviation
    sigma_w = np.sqrt(
        np.average((anchored - best_tau) ** 2, weights=np.abs(anchored - best_tau) + 1e-8)
    )
    delta = kappa * sigma_w

    pos, neg, unc = set(), set(), set()
    for concept, sc in scores.items():
        if sc > best_tau + delta:
            pos.add(concept)
        elif sc < best_tau - delta:
            neg.add(concept)
        else:
            unc.add(concept)

    return pos, neg, unc
```

## Best Practices

- **Do:** Run at least 3-5 prompt template variations per concept when collecting VLM confidence scores. Single-template scores are noisy; ensemble averaging is what makes the partitioning reliable.
- **Do:** Keep the concept vocabulary domain-specific and bounded. PolarMem's verification cost is O(items x concepts x templates). A vocabulary of 50-200 concepts is practical; 10,000 is not.
- **Do:** Include anchor priors `{0.0, 1.0}` when the concept set per item is small (< 10 scores). Without anchors, Otsu's method can produce degenerate thresholds.
- **Do:** Expose `HAS`/`NOT_HAS` edges in the agent's reasoning trace for auditability. The entire point is verifiability.
- **Avoid:** Using PolarMem for open-ended creative retrieval where "wrong but interesting" results are valuable. The logic-dominant paradigm aggressively filters, which kills serendipity.
- **Avoid:** Setting `kappa` too low (< 0.5). A narrow uncertain band means more concepts get polarized, increasing false-positive negation edges. Start with `kappa=1.0` and tune downward only if recall is too low.

## Error Handling

- **Degenerate Otsu threshold (all scores cluster together):** The anchor regularization handles this, but if the VLM returns near-identical scores for all concepts, the threshold becomes meaningless. Detect this condition (inter-class variance near zero) and fall back to a fixed threshold of 0.5 or skip partitioning for that item entirely.
- **VLM returns inconsistent scores across templates:** If the standard deviation across templates for a single concept exceeds 0.3, flag the concept as unreliable and assign it to the uncertain bin regardless of its mean score.
- **Graph becomes too sparse (most concepts uncertain):** Lower `kappa` or improve template quality. If more than 80% of concepts land in the uncertain bin, the polarization is not working and you are effectively running vanilla dense retrieval.
- **Latency from ensemble verification:** The offline indexing step is O(N x C x T) VLM calls (N items, C concepts, T templates). Parallelize across GPUs and batch inference. This is a one-time indexing cost, not a per-query cost.

## Limitations

- **Requires a concept vocabulary upfront.** PolarMem cannot verify concepts it was not asked about. If a relevant concept is missing from the vocabulary, it gets no edges and falls through to vanilla similarity matching.
- **Indexing cost scales multiplicatively.** For large corpora with broad concept vocabularies, the offline verification step can be expensive (10K images x 100 concepts x 5 templates = 5M VLM calls).
- **Diminishing returns on strong models.** The paper shows that on powerful VLMs (GPT-4o-class), PolarMem's gains on general reasoning benchmarks are marginal or slightly negative. The technique is most impactful on mid-tier models and retrieval-heavy tasks.
- **Static memory.** The current design builds the graph offline and queries it read-only. It does not support online updates where new evidence revises existing negation edges. Implementing incremental updates requires re-running verification for affected nodes.
- **Binary logic only.** The polarization is three-state (positive/negative/uncertain), not graded. It cannot express "somewhat likely present" -- that nuance is collapsed into the uncertain bin.

## Reference

**Paper:** [PolarMem: A Training-Free Polarized Latent Graph Memory for Verifiable Multimodal Agents](https://arxiv.org/abs/2602.00415v1) (Chen et al., 2026). Focus on Section 3 for the distributional partitioning algorithm, Section 4 for the polarized graph topology, and Algorithm 1 for the lexicographic retrieval protocol. Code: [github.com/czs-ict/PolarMem](https://github.com/czs-ict/PolarMem).