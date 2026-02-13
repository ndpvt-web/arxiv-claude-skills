---
name: "viola-video-in-context-learning"
description: "Apply the VIOLA framework for label-efficient in-context learning on video or multimodal data. Uses density-uncertainty-weighted sampling to select the most informative examples for annotation, builds hybrid pools mixing ground-truth and pseudo-labels, and uses confidence-aware retrieval and prompting to maximize few-shot performance. Trigger phrases: 'few-shot video classification', 'label-efficient in-context learning', 'select best examples to annotate', 'VIOLA sampling', 'confidence-aware prompting', 'active learning for LLM in-context examples'"
---

# VIOLA: Video In-Context Learning with Minimal Annotations

This skill teaches Claude to implement the VIOLA framework — a label-efficient in-context learning pipeline that maximizes multimodal LLM performance when only a small annotation budget is available. Rather than randomly selecting examples for labeling, VIOLA uses density-uncertainty-weighted sampling to pick the most representative and informative samples, generates confident pseudo-labels for the rest, and constructs prompts that explicitly signal label reliability to the model. The technique generalizes beyond video to any domain where you have many unlabeled samples but can only afford to label a few.

## When to Use

- When the user needs to classify or analyze video/image data with a multimodal LLM but has very few labeled examples (5-100)
- When building an in-context learning pipeline and the user asks how to select which examples to annotate from a large unlabeled pool
- When the user wants to combine a small labeled set with pseudo-labeled data for retrieval-augmented few-shot prompting
- When designing active-learning or sample-selection strategies for LLM-based classification
- When the user asks how to make in-context learning robust to noisy or uncertain demonstration labels
- When implementing retrieval-augmented generation (RAG) for classification tasks and wanting to factor in label confidence

## Key Technique

**The core problem:** Standard in-context learning picks demonstrations randomly or by similarity from a fully-labeled pool. In low-resource settings, you can't label everything. Naive active-learning strategies (pure uncertainty sampling or pure diversity sampling) select outliers that waste your annotation budget.

**VIOLA's solution has three interlocking parts:**

1. **Density-Uncertainty-Weighted Sampling** selects which samples to send for expert annotation. Fit a Gaussian Mixture Model (GMM) with K components (K = annotation budget B) over the embedding space of all unlabeled data. For each cluster, select the single sample that maximizes `S(x) = gamma^(1-lambda) * (1 - c_zero)^lambda`, where `gamma` is the GMM posterior (density/representativeness) and `c_zero` is the zero-shot confidence (certainty — so `1 - c_zero` is uncertainty). The parameter `lambda` in [0,1] balances representativeness vs. informativeness. This guarantees each selected sample is central to its cluster (not an outlier) yet hard for the model (high information value).

2. **Hybrid Pool Construction** augments the small labeled set with pseudo-labels. Run the MLLM on all remaining unlabeled samples using the newly-labeled examples as in-context demonstrations. Keep only pseudo-labels whose confidence exceeds the 95th percentile threshold. The hybrid pool is then `D_H = D_labeled ∪ D_pseudo`, where each entry stores (sample, label, confidence_score) — confidence is 1.0 for ground-truth items.

3. **Confidence-Aware Retrieval and Prompting** selects demonstrations from the hybrid pool at inference. Rank candidates by a composite score: `r = sim(x_candidate, x_query)^(1-tau) * confidence^tau`, where `tau` balances relevance vs. reliability. In the prompt, ground-truth examples are marked as such ("Ground-truth"), while pseudo-labels include their confidence percentage ("Pseudo-label with 87% confidence"). This lets the model internally weight how much to trust each demonstration.

## Step-by-Step Workflow

1. **Embed all samples.** Encode every unlabeled sample into a fixed-dimensional embedding using a suitable encoder (e.g., InternVideo2 for video, CLIP for images, or a sentence transformer for text). Store embeddings in a matrix for clustering.

2. **Obtain zero-shot confidence scores.** Run the target MLLM on each sample with zero demonstrations to get a baseline prediction and confidence. Confidence is the minimum token probability across the predicted output sequence (conservative estimate).

3. **Fit a GMM with K = B components.** Set the number of Gaussian mixture components equal to your annotation budget B. Fit the GMM on the embedding matrix using EM. Each component represents a region of the data distribution.

4. **Select one sample per cluster using the density-uncertainty score.** For each GMM cluster k, compute `S_k(x_i) = gamma_ik^(1-lambda) * (1 - c_zero_i)^lambda` for all samples. Select the single highest-scoring sample from each cluster. This yields exactly B samples to annotate. Use `lambda = 0.5` as a default starting point.

5. **Acquire expert labels for the B selected samples.** Send only these samples for human annotation. Store them as the labeled set `D_L` with confidence = 1.0.

6. **Generate pseudo-labels for remaining samples.** Use the MLLM with the B labeled samples as in-context demonstrations to predict labels for all remaining unlabeled samples. Record the confidence score for each prediction.

7. **Filter pseudo-labels by confidence threshold.** Compute the 95th percentile of all pseudo-label confidence scores. Discard any pseudo-label below this threshold. The surviving pseudo-labels form `D_P`.

8. **Construct the hybrid demonstration pool.** Merge: `D_H = D_L ∪ D_P`. Each entry stores (sample_embedding, label, confidence, source_type).

9. **At inference, retrieve demonstrations using confidence-aware scoring.** For a new query, compute `r_i = sim(embed_query, embed_i)^(1-tau) * confidence_i^tau` for all pool entries. Select the top-K (e.g., K=8). Sort the selected demonstrations: pseudo-labels first (ascending similarity), then ground-truth (ascending similarity), so the most relevant ground-truth appears closest to the query in the prompt.

10. **Format the prompt with confidence-aware labels.** For ground-truth demonstrations, write: "The correct label is [LABEL]. (Ground-truth)". For pseudo-labeled demonstrations, write: "The predicted label is [LABEL]. (Pseudo-label with [X]% confidence)". Append the query sample and ask the model to predict.

## Concrete Examples

**Example 1: Industrial Defect Classification with 20 Labels**

```
User: I have 500 images of manufacturing parts. I can only afford to label 20
of them. I need to classify defects (scratch, dent, crack, OK) using a
vision-language model. How should I pick which 20 to label?

Approach:
1. Embed all 500 images with CLIP ViT-L/14.
2. Run GPT-4o zero-shot on all 500 to get baseline confidence scores.
3. Fit a 20-component GMM on the 500 embeddings.
4. For each of the 20 clusters, select the image maximizing:
   S = gamma^0.5 * (1 - c_zero)^0.5
   This picks images that are central to their cluster AND hard for the model.
5. Send these 20 images to a quality inspector for labeling.
6. Use the 20 labeled images as ICL demos to pseudo-label the other 480.
7. Keep only pseudo-labels above 95th-percentile confidence (~top 24 images).
8. Build a hybrid pool of 20 ground-truth + ~24 pseudo-labeled images.
9. At inference, retrieve 8 demos per query using confidence-aware scoring
   with tau=0.3 (favor similarity over confidence since domain is narrow).

Output (sample selection summary):
  Cluster  | Selected Image | GMM Posterior | Zero-shot Conf | Score
  ---------|----------------|---------------|----------------|------
  1        | img_042.jpg    | 0.91          | 0.23           | 0.83
  2        | img_317.jpg    | 0.87          | 0.31           | 0.77
  ...
  20       | img_189.jpg    | 0.79          | 0.18           | 0.80

  Annotation budget: 20 images (4% of dataset)
  Pseudo-labels retained: 26 (above 95th percentile confidence)
  Total hybrid pool: 46 demonstrations
```

**Example 2: Surgical Phase Recognition with Limited Expert Time**

```
User: I have 200 surgical video clips across 8 phases. A surgeon can annotate
30 clips maximum. Build me an in-context learning pipeline.

Approach:
1. Extract embeddings using InternVideo2 (sample 1 fps, cap at 32 frames).
2. Get zero-shot predictions from Qwen2-VL-7B for all 200 clips.
3. Fit a 30-component GMM on the video embeddings.
4. Select 30 clips using density-uncertainty sampling (lambda=0.6,
   slightly favoring uncertainty since surgical phases are subtle).
5. Have the surgeon label the 30 selected clips.
6. Pseudo-label remaining 170 clips using the 30 as ICL examples.
7. Filter: keep pseudo-labels above 95th percentile confidence.
8. At inference, retrieve 8 demonstrations per query:
   r_i = sim(q, x_i)^0.6 * confidence_i^0.4
9. Format prompt with confidence tags:
   - "Phase: Dissection (Ground-truth)" for expert-labeled
   - "Phase: Coagulation (Pseudo-label with 92% confidence)" for pseudo-labeled
10. Arrange: pseudo-labels first, ground-truth last (closest to query).

Output (prompt structure):
  [Demo 1] Pseudo-label: Phase = Preparation (89% confidence)
  [Demo 2] Pseudo-label: Phase = Dissection (93% confidence)
  ...
  [Demo 7] Ground-truth: Phase = Coagulation
  [Demo 8] Ground-truth: Phase = Dissection
  [Query] What surgical phase is shown in this video?
```

**Example 3: Python Implementation of the Sampling Algorithm**

```python
# User: Implement the density-uncertainty-weighted sampling from VIOLA.

import numpy as np
from sklearn.mixture import GaussianMixture

def viola_select_samples(embeddings: np.ndarray,
                         zero_shot_confidences: np.ndarray,
                         budget: int,
                         lam: float = 0.5) -> list[int]:
    """Select `budget` samples using VIOLA density-uncertainty weighting.

    Args:
        embeddings: (N, D) array of sample embeddings.
        zero_shot_confidences: (N,) array of zero-shot confidence scores in [0,1].
        budget: Number of samples to select (B).
        lam: Balance between representativeness (0) and informativeness (1).

    Returns:
        List of selected sample indices.
    """
    gmm = GaussianMixture(n_components=budget, random_state=42)
    gmm.fit(embeddings)
    # gamma_ik: posterior probability of sample i belonging to cluster k
    posteriors = gmm.predict_proba(embeddings)  # (N, B)

    selected = []
    for k in range(budget):
        gamma_k = posteriors[:, k]                     # density term
        uncertainty = 1.0 - zero_shot_confidences      # informativeness term
        scores = (gamma_k ** (1 - lam)) * (uncertainty ** lam)
        # Exclude already-selected samples
        for idx in selected:
            scores[idx] = -1.0
        selected.append(int(np.argmax(scores)))

    return selected


def confidence_aware_retrieval(query_emb: np.ndarray,
                                pool_embs: np.ndarray,
                                pool_confidences: np.ndarray,
                                top_k: int = 8,
                                tau: float = 0.3) -> list[int]:
    """Retrieve top-K demos using VIOLA's confidence-aware scoring.

    Args:
        query_emb: (D,) query embedding.
        pool_embs: (M, D) hybrid pool embeddings.
        pool_confidences: (M,) confidence scores (1.0 for ground-truth).
        top_k: Number of demonstrations to retrieve.
        tau: Balance between similarity (0) and confidence (1).

    Returns:
        Indices into the pool, ordered for prompting (pseudo first, GT last).
    """
    sims = pool_embs @ query_emb / (
        np.linalg.norm(pool_embs, axis=1) * np.linalg.norm(query_emb) + 1e-8
    )
    scores = (sims ** (1 - tau)) * (pool_confidences ** tau)
    top_indices = np.argsort(scores)[-top_k:]

    # Sort: pseudo-labels first (ascending sim), ground-truth last (ascending sim)
    gt_mask = pool_confidences[top_indices] == 1.0
    pseudo_idxs = top_indices[~gt_mask]
    gt_idxs = top_indices[gt_mask]
    pseudo_sorted = pseudo_idxs[np.argsort(sims[pseudo_idxs])]
    gt_sorted = gt_idxs[np.argsort(sims[gt_idxs])]

    return list(pseudo_sorted) + list(gt_sorted)
```

## Best Practices

- **Do:** Use `lambda = 0.5` and `tau = 0.3` as sensible defaults. Increase `lambda` when classes are subtle or overlapping (need more uncertain samples). Increase `tau` when your pseudo-labels are noisy (prioritize reliability over similarity).
- **Do:** Always mark label provenance in the prompt. The explicit "(Ground-truth)" vs "(Pseudo-label with X% confidence)" tags measurably improve model reasoning about demonstration quality.
- **Do:** Sort demonstrations so ground-truth examples appear closest to the query in the prompt. Models attend more strongly to nearby context, so placing your most reliable labels last maximizes their influence.
- **Do:** Apply the 95th-percentile confidence filter strictly when building the pseudo-label set. Including low-confidence pseudo-labels degrades performance below even the labeled-only baseline.
- **Avoid:** Using pure diversity sampling (e.g., k-medoids) or pure uncertainty sampling alone. Diversity sampling picks outliers in sparse regions; uncertainty sampling clusters around decision boundaries and misses easy-but-representative regions.
- **Avoid:** Treating pseudo-labels as equivalent to ground truth. The entire point of confidence-aware prompting is to let the model discount unreliable demonstrations. Removing the confidence tags collapses performance.

## Error Handling

- **GMM fails to converge:** Reduce the number of components or increase `max_iter`. If the embedding space is very high-dimensional, apply PCA to 128-256 dimensions before fitting the GMM.
- **Zero-shot confidence is uniformly low:** This is actually desirable — it means most samples are informative. The density term will dominate selection, ensuring representativeness. If confidence is uniformly *high*, increase `lambda` to emphasize the small uncertainty differences.
- **Too few pseudo-labels pass the 95th-percentile filter:** The filtering is working correctly by excluding unreliable predictions. If the pseudo-label pool is very small, fall back to using only the labeled set for retrieval. Do not lower the threshold below the 90th percentile.
- **Embedding model mismatch:** The embedding model used for clustering and retrieval should capture task-relevant features. For video tasks, use a video encoder (InternVideo2, VideoCLIP). For image tasks, use CLIP. For text, use a sentence transformer. A poor embedding model will produce meaningless clusters.
- **Budget exceeds natural cluster count:** If the GMM has more components than natural clusters, some components will be near-duplicates. This is acceptable — you'll still get good coverage but with some redundancy in annotation.

## Limitations

- The framework assumes the MLLM can perform zero-shot inference to produce initial confidence scores. If the model has zero capability on the domain (e.g., highly specialized medical imagery), the uncertainty estimates will be uninformative and sampling degrades to density-only.
- Confidence-aware prompting requires the model to understand and use the reliability tags. Smaller or less capable models may ignore these textual cues, reducing the benefit of the hybrid pool.
- The technique is designed for classification-style tasks (discrete labels). For open-ended generation tasks (captioning, VQA with free-form answers), the confidence scoring and pseudo-labeling steps need adaptation.
- The 95th-percentile threshold is a heuristic. For highly imbalanced datasets, rare classes may have systematically lower confidence, causing their pseudo-labels to be disproportionately filtered out.
- In-context window limits constrain the number of demonstrations. With 8 video demonstrations at 16 frames each, the context can fill up quickly. This bounds how much of the hybrid pool can be used per query.

## Reference

**Paper:** [VIOLA: Towards Video In-Context Learning with Minimal Annotations](https://arxiv.org/abs/2601.15549v1) — Fujii, Saito, Hachiuma (2026). Look for Section 3 (method) for the density-uncertainty sampling formula (Eq. 8), confidence-aware retrieval scoring (Eq. 11), and the prompt formatting strategy with reliability tags.