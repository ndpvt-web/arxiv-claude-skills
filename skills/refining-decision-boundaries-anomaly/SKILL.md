---
name: "refining-decision-boundaries-anomaly"
description: >
  Build anomaly detection systems that refine decision boundaries using similarity search
  in learned feature spaces. Implements the SDA2E framework: a Sparse Dual Adversarial
  Attention-based AutoEncoder with similarity-guided active learning for imbalanced datasets.
  Trigger phrases: "detect anomalies in imbalanced data", "build an anomaly detector with
  active learning", "APT detection pipeline", "rank anomalies with minimal labels",
  "similarity-guided anomaly detection", "sparse autoencoder for cybersecurity"
---

# Refining Decision Boundaries in Anomaly Detection Using Similarity Search

This skill enables Claude to build anomaly detection systems based on the SDA2E framework,
which combines a sparse dual adversarial autoencoder with similarity-guided active learning
to detect rare anomalies in highly imbalanced datasets. The core insight is that decision
boundaries between normal and anomalous data can be iteratively sharpened by searching the
learned feature space for points similar to known normals or known anomalies, then selectively
querying an oracle -- reducing required labeled data by up to 80% versus passive labeling.

## When to Use

- When the user needs to detect rare events (fraud, intrusions, APTs) in datasets where anomalies are < 1-5% of samples
- When the user asks to build an anomaly ranking system that minimizes labeling effort
- When implementing cybersecurity threat detection (e.g., DARPA Transparent Computing scenarios, network log analysis)
- When the user has high-dimensional tabular or event data with severe class imbalance and wants unsupervised-to-active-learning progression
- When the user asks for a similarity-based approach to refine anomaly detection beyond simple threshold tuning
- When building a system that needs to rank suspicious items for analyst review with nDCG-optimized ordering

## Key Technique

**SDA2E** (Sparse Dual Adversarial Attention-based AutoEncoder) trains two autoencoders adversarially: a Generator (G) that reconstructs inputs through a sparse bottleneck, and a Discriminator (D) that assigns energy scores to distinguish real from reconstructed samples. An attention module learns per-feature importance weights, gating irrelevant dimensions. The bottleneck uses ReLU activation with a KL-divergence sparsity penalty (target ~10% neuron activation), producing compact binary-like latent codes. The anomaly score for any point is its reconstruction error: `Ascore(x) = ||x - G(x)||^2`. Normal points reconstruct well; anomalies don't.

**Similarity-Guided Active Learning** is what makes this framework label-efficient. After initial training on a small labeled seed set, three strategies refine the model: (1) **Normal-like expansion** finds unlabeled points whose latent representations are similar to labeled normals (above the 80th percentile of the similarity distribution) and adds them to the training set, improving reconstruction fidelity for the normal class. (2) **Anomaly-like prioritization** finds unlabeled points similar to labeled anomalies and promotes them to the top of the ranked output list, boosting ranking accuracy without retraining. (3) **Hybrid** applies both simultaneously. The similarity search uses **SIM_NM1** (Normalized Matching 1s), a metric tailored for sparse binary embeddings that counts matching active dimensions normalized by total non-zero positions across both vectors.

**Why this works better**: Traditional active learning queries the most uncertain points. SDA2E instead exploits the geometric structure of the feature space -- expanding known clusters of normality and anomaly through similarity propagation. This means each oracle query yields information not just about one point, but about its neighborhood.

## Step-by-Step Workflow

1. **Prepare the imbalanced dataset.** Load data into a numerical matrix. Normalize features to [0, 1] or standardize. Identify categorical columns and one-hot encode them. Record the class distribution -- this framework targets scenarios where anomalies are < 5% of data.

2. **Build the SDA2E autoencoder pair.** Construct the Generator autoencoder (encoder: input_dim -> 256 -> 128 -> k, decoder: k -> 128 -> 256 -> input_dim) and Discriminator autoencoder (same architecture). Add the attention module: a single dense layer with sigmoid activation producing per-feature weights in [0, 1]. Set bottleneck dimension k < input_dim (typically 32-64). Use ReLU in the bottleneck for sparsity.

3. **Define the composite loss function.** Implement four loss terms: reconstruction loss (MSE), adversarial loss (energy-based margin loss with margin m), sparsity loss (KL divergence between average bottleneck activation and target sparsity rho=0.1), and attention L1 regularization. Combine as: `L_G = L_recon + alpha * L_adv_G + beta * L_sparse + gamma * L_attn`. Train D with: `L_D = L_adv_D + delta * L_sparse_D`.

4. **Train with alternating optimization.** For each mini-batch, update D first (freeze G), then update G (freeze D). Attention parameters train jointly with G. Use Adam optimizer. Start with a small labeled seed set (even 1-5% of data suffices). Train until reconstruction loss stabilizes.

5. **Compute anomaly scores and initial ranking.** Pass all data through the trained Generator. Compute reconstruction error `||x - G(x)||^2` for each point. Rank all points by descending error -- high error = likely anomaly.

6. **Generate sparse binary embeddings.** Extract bottleneck activations for all points. Binarize: set values > threshold (e.g., median activation) to 1, rest to 0. These sparse binary vectors are the feature-space representations used for similarity search.

7. **Implement the SIM_NM1 similarity measure.** For two binary vectors a, b: `SIM_NM1(a, b) = |{i : a_i = 1 AND b_i = 1}| / |{i : a_i = 1 OR b_i = 1}|`. This is essentially the Jaccard index over active bits, optimized for sparse vectors where most entries are 0.

8. **Apply similarity-guided active learning (iterative loop).** Select top-ranked unlabeled points (above 80th percentile of reconstruction error). Query the oracle for labels. Then apply one of three strategies:
   - **Normal-like expansion**: For each newly labeled normal, find all unlabeled points with `SIM_NM1 >= 80th percentile threshold`, add them to training set, retrain G.
   - **Anomaly-like prioritization**: For each newly labeled anomaly, find similar unlabeled points, promote them to top of ranked list.
   - **Hybrid**: Do both. Use this as the default strategy.

9. **Iterate and re-rank.** After retraining (if normal-like expansion was used), recompute anomaly scores and re-rank. Repeat steps 8-9 for a fixed number of active learning rounds (typically 5-10) or until ranking stabilizes (nDCG change < 0.01).

10. **Output the final ranked anomaly list.** Return all data points sorted by anomaly score with confidence indicators. Report nDCG if ground truth is available. Flag the top-k items for human review.

## Concrete Examples

**Example 1: Network Intrusion Detection on DARPA-style Logs**

User: "I have a CSV of 500K network connection records with 40 features. Only ~200 are known APT-related. Build an anomaly detection pipeline that ranks suspicious connections."

Approach:
1. Load CSV, normalize numerical features (bytes_sent, duration, etc.), one-hot encode protocol types
2. Build SDA2E with input_dim=80 (after encoding), bottleneck k=32
3. Train on full dataset unsupervised, then seed with 10 labeled normals and 5 labeled anomalies
4. Compute reconstruction errors, rank all 500K connections
5. Extract binary embeddings from bottleneck, compute SIM_NM1 pairwise against labeled set
6. Run hybrid active learning: expand normal training set via similarity, prioritize anomaly-similar points
7. After 5 rounds querying ~50 points total, produce final ranking

Output:
```python
# Final ranked output (top 10 of 500K)
rank | connection_id | anomaly_score | strategy_flag
1    | conn_389201   | 0.0847        | anomaly_similar (to labeled APT conn_44)
2    | conn_102334   | 0.0831        | anomaly_similar (to labeled APT conn_44)
3    | conn_445002   | 0.0789        | high_recon_error
4    | conn_221890   | 0.0756        | anomaly_similar (to labeled APT conn_12)
...
# nDCG@100: 0.94 | nDCG@all: 0.87
# Total labels queried: 65 (0.013% of dataset)
```

**Example 2: Fraud Detection in Financial Transactions**

User: "I need to detect fraudulent transactions. I have 1M records, ~0.1% are fraud. I can only afford to manually review 500 cases. Help me pick the best 500."

Approach:
1. Preprocess transaction features (amount, time_delta, merchant_category, etc.)
2. Build SDA2E, train unsupervised on all 1M records
3. Initial ranking by reconstruction error gives a rough top-500
4. From that top-500, oracle-label 50 (5 rounds x 10 per round)
5. Apply anomaly-like prioritization: for each confirmed fraud, find similar unlabeled transactions via SIM_NM1 in the binary embedding space
6. Re-rank after each round, surfacing fraud-similar transactions higher

Output:
```python
# Active learning progression
Round | Labels_queried | Confirmed_fraud | nDCG@500
0     | 0              | -               | 0.61    # unsupervised baseline
1     | 10             | 3               | 0.72
2     | 20             | 7               | 0.81
3     | 30             | 9               | 0.88
4     | 40             | 11              | 0.92
5     | 50             | 14              | 0.95
# Final top-500 list captures 94% of all fraud vs 61% unsupervised
```

**Example 3: Implementing SIM_NM1 for Sparse Binary Vectors**

User: "Show me how to implement the similarity measure for sparse binary embeddings."

```python
import numpy as np
from scipy.sparse import issparse

def sim_nm1(a: np.ndarray, b: np.ndarray) -> float:
    """Normalized Matching 1s similarity for sparse binary vectors.
    Equivalent to Jaccard index over active (non-zero) positions."""
    matching_ones = np.sum((a == 1) & (b == 1))
    union_ones = np.sum((a == 1) | (b == 1))
    if union_ones == 0:
        return 0.0
    return matching_ones / union_ones

def sim_nm1_batch(query: np.ndarray, reference_set: np.ndarray) -> np.ndarray:
    """Compute SIM_NM1 between one query vector and a set of reference vectors.
    Optimized for batch computation on sparse binary matrices."""
    query_expanded = query.reshape(1, -1)
    matching = np.sum((query_expanded == 1) & (reference_set == 1), axis=1)
    union = np.sum((query_expanded == 1) | (reference_set == 1), axis=1)
    union[union == 0] = 1  # avoid division by zero
    return matching / union

def find_similar(query, reference_set, percentile_threshold=80):
    """Return indices of reference points similar to query above adaptive threshold."""
    sims = sim_nm1_batch(query, reference_set)
    threshold = np.percentile(sims[sims > 0], percentile_threshold)
    return np.where(sims >= threshold)[0], sims
```

## Best Practices

- **Do:** Start with unsupervised pretraining before any active learning. The initial reconstruction quality sets the floor for all subsequent refinement.
- **Do:** Use the hybrid strategy (Strategy 3) as the default. It consistently outperforms either normal-like expansion or anomaly-like prioritization alone across diverse datasets.
- **Do:** Set sparsity target rho between 0.05 and 0.15. Too sparse (< 0.05) loses discriminative power; too dense (> 0.2) makes SIM_NM1 less effective.
- **Do:** Use adaptive percentile thresholds (80th percentile of similarity distribution) rather than fixed thresholds. This automatically adjusts to dataset-specific distributions.
- **Avoid:** Training the autoencoder only on labeled normals. SDA2E benefits from seeing the full unlabeled distribution during initial training -- the adversarial discriminator needs both modes.
- **Avoid:** Querying more than 20% of the dataset through active learning. The framework's value comes from label efficiency -- if you're labeling > 20%, switch to supervised methods.

## Error Handling

- **Collapsed latent space (all embeddings identical):** Reduce the sparsity penalty beta or increase bottleneck dimension k. The adversarial loss may also need a larger margin m.
- **Active learning doesn't improve nDCG:** Check that the binary embedding threshold isn't too aggressive. If > 95% of bits are 0, the SIM_NM1 metric can't differentiate points. Lower the binarization threshold.
- **Oracle returns only normals (no anomalies found in top-ranked):** Switch from anomaly-like prioritization to normal-like expansion exclusively for 2-3 rounds. Expanding the normal representation will push true anomalies higher in subsequent rankings.
- **Memory issues with pairwise similarity on large datasets:** Compute SIM_NM1 in batches or use approximate nearest neighbor search (e.g., FAISS with binary vectors) for datasets > 100K points.
- **Feature attention collapses to uniform weights:** Increase the L1 regularization gamma on the attention module. Attention should be selective -- if all weights are ~0.5, the gating isn't learning.

## Limitations

- **Requires at least some labeled seed data** for the active learning loop. Purely unsupervised mode (no oracle) gives baseline reconstruction-error ranking but misses the decision boundary refinement that is the core contribution.
- **Tabular/structured data focus.** SDA2E's architecture is designed for fixed-dimensional feature vectors. For sequence data (logs, packet captures), you must first extract fixed-length feature vectors.
- **Binary embedding loses information.** The binarization step trades representational fidelity for similarity-search efficiency. For datasets where subtle continuous variations matter, consider using cosine similarity on the raw latent vectors instead of SIM_NM1 on binary vectors.
- **Not a streaming detector.** The framework assumes batch processing with iterative retraining. For real-time anomaly detection, use the trained model for scoring but the active learning loop must run offline.
- **Adversarial training instability.** Like all GAN-adjacent architectures, the dual autoencoder can suffer from mode collapse or oscillation. Monitor both G and D losses during training.

## Reference

**Paper:** "Refining Decision Boundaries In Anomaly Detection Using Similarity Search Within the Feature Space" -- Benabderrahmane, Valtchev, Cheney, Rahwan (2026). [arXiv:2602.02925](https://arxiv.org/abs/2602.02925v1)

**Key insight to extract:** Section on the three active learning strategies and their similarity-guided query selection mechanism -- this is what distinguishes SDA2E from standard autoencoder-based anomaly detectors. The 80th-percentile adaptive thresholding for SIM_NM1 is critical for practical deployment.