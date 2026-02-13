---
name: "do-reasoning-enhance-embedding"
description: |
  Analyze whether reasoning-tuned LLMs improve embedding quality using the HRSA framework
  (Hierarchical Representation Similarity Analysis). Diagnose representation differences
  between base and fine-tuned models across three levels: representation, geometry, and function.
  Guide embedding model initialization decisions and avoid wasting compute on reasoning backbones.

  Trigger phrases:
  - "Should I use a reasoning model as my embedding backbone?"
  - "Compare base vs reasoning model embeddings"
  - "Analyze representation similarity between two models"
  - "Does RLVR improve embeddings?"
  - "Diagnose why my fine-tuned embedding model isn't better"
  - "HRSA analysis of model representations"
---

# Reasoning Models as Embedding Backbones: HRSA Analysis Skill

This skill enables Claude to apply the Hierarchical Representation Similarity Analysis (HRSA) framework from Chan et al. (2026) to help users make informed decisions about embedding model initialization. The core finding: RLVR-tuned reasoning models (e.g., DeepSeek-R1, Qwen-reasoning variants) produce **no consistent embedding quality improvement** over base models after contrastive training. HRSA provides a three-level diagnostic framework (representation, geometry, function) to verify this and explain why, through a phenomenon called **Manifold Realignment** where contrastive learning erases the representational differences RLVR introduced.

## When to Use

- When a user asks whether to use a reasoning-tuned model (DeepSeek-R1, Qwen3-reasoning, etc.) as an embedding backbone
- When comparing two model checkpoints to understand if fine-tuning changed the representation space meaningfully
- When diagnosing why a supposedly better backbone didn't improve embedding benchmark scores (MTEB, BRIGHT)
- When deciding between base vs. SFT vs. RLVR model variants for embedding initialization
- When implementing a representation similarity analysis pipeline to compare any two encoder/decoder models
- When a user wants to understand whether their fine-tuning changed local geometry, global geometry, or just rotated the coordinate basis

## Key Technique: HRSA and Manifold Realignment

**HRSA** decomposes model similarity into three hierarchical levels, each answering a different question:

1. **Representation Level** — Are the embedding axes aligned? Measured via (a) *Dimension-Wise Correlation*: per-dimension cosine similarity `rho_j = X_j^T Y_j / (||X_j|| ||Y_j||)` averaged across dimensions, and (b) *Orthogonal Procrustes Analysis*: finds the best orthogonal rotation `O* = argmin ||XO - Y||^2_F` and examines the inverse row entropy of O* (high entropy = pure rotation/drift; low entropy approaching a permutation matrix = axis-aligned correspondence).

2. **Geometry Level** — Is the manifold shape preserved? Measured via (a) *Linear CKA* (Centered Kernel Alignment): compares Gram matrices `K_X = XX^T` using HSIC to assess global shape preservation, and (b) *k-NN Overlap*: Jaccard index of cosine-based k-nearest-neighbor sets, capturing local neighborhood fidelity. RLVR preserves global geometry (CKA ~1.0) but irreversibly reorganizes local neighborhoods (k-NN overlap drops to ~0.45 at k=5).

3. **Function Level** — Do the representations support the same downstream decisions? Measured via *Cross-Model Linear Probes*: train a linear classifier on model A's representations, evaluate on model B's. High cross-probe accuracy means the decision boundaries are interchangeable.

**Manifold Realignment** is the key discovery: contrastive training drives base-initialized and RLVR-initialized models toward the same final representation. The coordinate basis drift introduced by RLVR is *reversible* (Procrustes entropy recovers from 0.16 to 0.86 after contrastive training), while local geometry changes are *irreversible* but functionally inconsequential. This contrasts sharply with SFT, which causes destructive anisotropic distortion of the global manifold and degrades linear readout — effects that persist through contrastive training and cause measurable quality loss (up to -12 points on MTEB).

## Step-by-Step Workflow

### For Decision Making: Should I Use a Reasoning Backbone?

1. **Identify the backbone pair.** Determine the base model and its RLVR-tuned variant (e.g., `Qwen2.5-1.5B` vs `Qwen2.5-1.5B-SRL-Zoo`, or `DeepSeek-R1-Distill-Qwen-1.5B`). Confirm the reasoning variant was trained via RLVR (not SFT — the implications differ drastically).

2. **Check the empirical verdict first.** For RLVR backbones: expect near-zero delta on MTEB and BRIGHT after identical contrastive training (typical range: -0.3 to +0.7 points). For SFT backbones: expect significant degradation (-5 to -12 points). Recommend the **base model** unless there's a specific reason to prefer the reasoning variant.

3. **If the user needs proof, implement HRSA Level 1 (Representation).** Extract hidden states from the final layer of both models on a shared evaluation corpus. Compute dimension-wise correlation and Procrustes alignment. Low Procrustes entropy (<0.3) indicates coordinate drift; this is expected from RLVR and is reversible.

4. **Implement HRSA Level 2 (Geometry).** Compute Linear CKA between the Gram matrices of both models' representations. Compute k-NN overlap at k=5, 10, 50. CKA near 1.0 with k-NN overlap around 0.4-0.5 is the RLVR signature (global shape preserved, local neighborhoods shuffled).

5. **Implement HRSA Level 3 (Function).** Train a linear probe (logistic regression) on classification labels using model A's representations. Evaluate on model B's representations. High cross-probe accuracy (>90% of within-model accuracy) confirms functional equivalence.

6. **Synthesize the diagnosis.** Present findings as a three-level summary table. If global geometry is preserved (CKA > 0.95) and cross-probes transfer well, contrastive training will realign the models — recommend the cheaper base model.

### For Implementation: Building the HRSA Pipeline

7. **Set up the evaluation corpus.** Use 5,000-10,000 diverse text samples. Extract final-layer hidden states using mean pooling with bidirectional attention (remove causal mask). Store as numpy arrays of shape `(n_samples, hidden_dim)`.

8. **Compute representation metrics.** For dimension-wise correlation: normalize each dimension vector and compute cosine similarity, then average. For Procrustes: use `scipy.linalg.orthogonal_procrustes`, then compute row-wise entropy of the resulting rotation matrix.

9. **Compute geometry metrics.** For Linear CKA: center the representations, compute `K_X = XX^T` and `K_Y = YY^T`, then `CKA = HSIC(K_X, K_Y) / sqrt(HSIC(K_X, K_X) * HSIC(K_Y, K_Y))`. For k-NN overlap: build cosine-similarity neighbor graphs for both models and compute mean Jaccard index across all points.

10. **Compute function metrics.** Split data into train/test. Train `sklearn.linear_model.LogisticRegression` on model A train representations with task labels. Evaluate on both model A and model B test representations. Report accuracy ratio.

## Concrete Examples

**Example 1: Deciding on an embedding backbone**

User: "I'm building a retrieval system. Should I use DeepSeek-R1-Distill-Qwen-1.5B instead of the base Qwen-1.5B for my embedding model?"

Approach:
1. Identify that DeepSeek-R1-Distill is an RLVR-tuned variant of Qwen
2. Cite the null effect: RLVR backbones show -0.26 to +0.67 delta on MTEB Code and -0.06 to +0.45 on BRIGHT after identical contrastive training
3. Explain Manifold Realignment — contrastive training erases the RLVR-introduced differences
4. Recommend the base Qwen-1.5B to save on inference cost during training data generation and avoid unnecessary complexity

Output:
```
Recommendation: Use the base Qwen-1.5B.

Reasoning (per Chan et al. 2026, HRSA analysis):
- RLVR preserves global manifold geometry (Linear CKA ≈ 1.0)
- Contrastive training realigns coordinate basis drift (Procrustes entropy: 0.16 → 0.86)
- Empirical delta after identical training: < 0.7 points on all benchmarks
- The reasoning capabilities don't transfer to embedding quality

Exception: If you need the reasoning model for hard-negative mining
or query generation during data preparation, that's a valid use —
but use the base model as the actual embedding backbone.
```

**Example 2: Diagnosing why an SFT model degraded embedding quality**

User: "I fine-tuned Qwen3-0.6B with supervised instruction data, then trained it as an embedding model. It's 8 points worse than starting from base. Why?"

Approach:
1. Identify this as an SFT (not RLVR) scenario — fundamentally different dynamics
2. Run HRSA Level 2: expect low CKA (anisotropic distortion of global geometry)
3. Run HRSA Level 3: expect degraded cross-model linear probes
4. Explain that SFT restructures the semantic landscape, unlike RLVR which preserves it

Output:
```
Diagnosis: SFT caused irreversible manifold restructuring.

HRSA findings for SFT vs RLVR:
┌──────────────────┬────────────────────┬──────────────────┐
│ Metric           │ SFT Effect         │ RLVR Effect      │
├──────────────────┼────────────────────┼──────────────────┤
│ Coordinate basis │ Destructive mixing │ Reversible drift │
│ Global geometry  │ Anisotropic warp   │ Preserved        │
│ Local geometry   │ Reorganized        │ Reorganized      │
│ Linear readout   │ Degraded           │ Transferred      │
│ Post-contrastive │ -5 to -12 pts     │ -0.3 to +0.7 pts│
└──────────────────┴────────────────────┴──────────────────┘

Fix: Use the base Qwen3-0.6B-Base as your embedding backbone.
SFT fundamentally restructures the representation manifold in ways
that contrastive learning cannot recover from. RLVR, by contrast,
only optimizes trajectories within the existing semantic landscape.
```

**Example 3: Implementing an HRSA comparison pipeline**

User: "I have two model checkpoints and want to know if fine-tuning changed the representations meaningfully. Can you help me build an analysis?"

Approach:
1. Write a Python script that extracts representations from both models
2. Implement all three HRSA levels
3. Generate a diagnostic report

Output:
```python
import numpy as np
from scipy.linalg import orthogonal_procrustes
from scipy.stats import entropy
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import accuracy_score
from sklearn.neighbors import NearestNeighbors

def hrsa_analysis(X, Y, labels=None, k=5):
    """Run full HRSA analysis on two representation matrices.
    X, Y: np.array of shape (n_samples, hidden_dim), mean-pooled final-layer states.
    labels: optional array for cross-probe evaluation.
    k: neighborhood size for k-NN overlap.
    """
    results = {}

    # Level 1: Representation — Dimension-wise correlation
    X_norm = X / (np.linalg.norm(X, axis=0, keepdims=True) + 1e-8)
    Y_norm = Y / (np.linalg.norm(Y, axis=0, keepdims=True) + 1e-8)
    dim_corr = np.mean(np.sum(X_norm * Y_norm, axis=0) / X.shape[0])
    results['dim_correlation'] = dim_corr

    # Level 1: Representation — Procrustes alignment
    X_c = X - X.mean(axis=0)
    Y_c = Y - Y.mean(axis=0)
    R, _ = orthogonal_procrustes(X_c, Y_c)
    row_entropies = np.array([entropy(np.abs(row) + 1e-10) for row in R])
    results['procrustes_inv_row_entropy'] = 1.0 - np.mean(row_entropies) / np.log(R.shape[1])

    # Level 2: Geometry — Linear CKA
    K_X = X_c @ X_c.T
    K_Y = Y_c @ Y_c.T
    hsic_xy = np.sum(K_X * K_Y)
    hsic_xx = np.sum(K_X * K_X)
    hsic_yy = np.sum(K_Y * K_Y)
    results['linear_cka'] = hsic_xy / (np.sqrt(hsic_xx * hsic_yy) + 1e-8)

    # Level 2: Geometry — k-NN Overlap
    nn_x = NearestNeighbors(n_neighbors=k, metric='cosine').fit(X)
    nn_y = NearestNeighbors(n_neighbors=k, metric='cosine').fit(Y)
    idx_x = nn_x.kneighbors(X, return_distance=False)
    idx_y = nn_y.kneighbors(Y, return_distance=False)
    overlaps = [len(set(a) & set(b)) / len(set(a) | set(b))
                for a, b in zip(idx_x, idx_y)]
    results['knn_overlap'] = np.mean(overlaps)

    # Level 3: Function — Cross-model linear probes
    if labels is not None:
        n = len(labels)
        split = int(0.8 * n)
        clf = LogisticRegression(max_iter=1000).fit(X[:split], labels[:split])
        results['probe_within'] = accuracy_score(labels[split:], clf.predict(X[split:]))
        results['probe_cross'] = accuracy_score(labels[split:], clf.predict(Y[split:]))
        results['probe_transfer_ratio'] = results['probe_cross'] / (results['probe_within'] + 1e-8)

    return results

def interpret_hrsa(results):
    """Interpret HRSA results and classify the fine-tuning type."""
    cka = results.get('linear_cka', 0)
    knn = results.get('knn_overlap', 0)
    transfer = results.get('probe_transfer_ratio', 1.0)

    if cka > 0.95 and transfer > 0.9:
        return "RLVR-like: Global geometry preserved. Contrastive training will realign. Use base model."
    elif cka < 0.85 or transfer < 0.8:
        return "SFT-like: Manifold restructured. Embedding quality likely degraded. Use base model."
    else:
        return "Ambiguous: Run full contrastive training comparison to confirm."
```

## Best Practices

**Do:**
- Always check whether a reasoning model was trained via RLVR or SFT before making initialization decisions — the effects are opposite
- Use the base model as the embedding backbone when the reasoning variant was trained via RLVR; save the reasoning model for data generation tasks (hard-negative mining, query synthesis)
- Run HRSA at all three levels before concluding two models are equivalent; CKA alone can miss local geometry changes
- Use at least 5,000 samples for stable CKA and k-NN estimates; fewer samples inflate variance on geometry metrics
- Center representations before computing CKA and Procrustes (subtract mean across samples)

**Avoid:**
- Assuming reasoning models produce better embeddings — this is the central null result of the paper
- Using SFT-tuned models as embedding backbones without testing; SFT causes -5 to -12 point degradation that persists through contrastive training
- Interpreting low dimension-wise correlation alone as evidence of different representations — coordinate basis drift is reversible under contrastive training
- Conflating local geometry changes (k-NN overlap) with functional changes (cross-probes); reorganized neighborhoods don't necessarily affect downstream tasks

## Error Handling

- **Misidentified training method:** If the user doesn't know whether a model used RLVR or SFT, run HRSA Level 2. CKA > 0.95 with k-NN ~0.45 suggests RLVR; CKA < 0.85 with degraded cross-probes suggests SFT.
- **Small evaluation corpus:** With fewer than 1,000 samples, CKA estimates become unstable. Warn the user and recommend bootstrapped confidence intervals.
- **Dimensionality mismatch:** If comparing models with different hidden dimensions, project both to the smaller dimension via PCA before computing HRSA metrics.
- **Memory issues with large Gram matrices:** For n > 50,000 samples, use minibatch CKA (compute on random subsets of 5,000 and average across 10 runs).

## Limitations

- The null effect was demonstrated on models up to 4B parameters. Larger models (70B+) may exhibit different dynamics — the finding may not extrapolate.
- HRSA assumes access to hidden states from both models on the same input corpus. This requires running inference on both models, which doubles compute.
- The contrastive training recipe used a specific configuration (batch 2048, lr 2e-5, temperature 0.02, 782 steps). Different recipes could yield different realignment dynamics.
- Manifold Realignment was observed with InfoNCE loss. Other contrastive objectives (triplet loss, cosine similarity loss) have not been tested.
- The paper focuses on text embedding. Multimodal embedding (CLIP-style) may behave differently since vision encoders have distinct manifold properties.

## Reference

**Paper:** Chan et al. (2026). "Do Reasoning Models Enhance Embedding Models?" [arXiv:2601.21192v1](https://arxiv.org/abs/2601.21192v1)
**Code:** [HKUST-KnowComp/Reasoning-Embedding](https://github.com/HKUST-KnowComp/Reasoning-Embedding)
**Key insight:** RLVR-tuned reasoning models yield no embedding quality improvement over base models because contrastive learning triggers Manifold Realignment — use HRSA's three-level analysis to verify this for any model pair.