---
name: "sparseeval-evaluation-sparse-optimization"
description: "Efficiently evaluate LLMs on benchmarks by selecting a small subset of anchor items via sparse optimization, reproducing full-benchmark rankings at a fraction of the cost. Use when: 'reduce evaluation cost for my LLM benchmark', 'select representative test items from a large dataset', 'rank models without running all benchmark samples', 'sparse subset selection for evaluation', 'find anchor items that represent my test suite', 'efficient model comparison on benchmarks'."
---

# SparseEval: Efficient LLM Evaluation via Sparse Optimization

This skill enables Claude to help users drastically reduce LLM evaluation costs by selecting a small, representative subset of benchmark items (called **anchors**) whose results can predict full-benchmark performance. The core technique from the ICLR 2026 paper *SparseEval* formulates benchmark reduction as a sparse optimization problem: construct a model-item performance matrix, discover its inherent sparsity via spectral clustering, select anchors through k-means initialization, optimize anchor weights with an MLP trained via gradient descent, and iteratively refine the anchor set using Anchor Importance Scores (AIS) and Candidate Importance Scores (CIS). The result: evaluate on ~100 items instead of thousands while maintaining Kendall's tau > 0.9 rank correlation with full evaluation.

## When to Use

- When the user wants to **reduce the number of benchmark items** they need to run inference on to rank or compare LLMs
- When building a **custom evaluation suite** and needs to select the most informative test items from a large pool
- When the user has a **model-item performance matrix** (rows = models, columns = test items, entries = correct/incorrect) and wants to find which items matter most
- When the user asks to **rank models cheaply** without full benchmark runs, using historical evaluation data from leaderboards
- When designing a **lightweight evaluation pipeline** for CI/CD or rapid model iteration where full benchmarks are too expensive
- When the user wants to **identify redundant test items** in an existing benchmark through sparsity analysis

## Key Technique

**The Sparsity Insight.** SparseEval starts from the observation that the binary model-item performance matrix (1 for correct, -1 for incorrect) across thousands of models is inherently sparse and clustered. When you compute cosine similarity between item vectors and apply spectral clustering, pronounced diagonal blocks emerge -- groups of items that models tend to get right or wrong together. This redundancy means a small subset of "anchor" items can represent the entire benchmark.

**MLP-Based Weight Optimization.** Given a set of anchor items, SparseEval trains a small MLP to learn weights that minimize the reconstruction loss between the weighted anchor scores and the true full-benchmark scores. The loss is `L = (1/M) * ||f(S_train * (1_M * W^T)) - S_train * W_a||_2` where `f` is the MLP, `S_train` is the performance matrix for training models, `W` encodes sparse anchor weights, and `W_a` is the uniform average. The MLP's nonlinearity captures complex item interactions that linear weighting misses -- deeper architectures significantly outperform linear baselines.

**Iterative Anchor Refinement.** After training, the method computes two scores: (1) **Anchor Importance Score (AIS)** = L1 norm of the gradient of the loss with respect to each anchor's column, measuring how much that anchor contributes to accuracy; (2) **Candidate Importance Score (CIS)** = absolute dot product of each non-anchor item's column with the residual error vector, measuring how much adding that item would reduce remaining error. Each refinement iteration drops the lowest-AIS anchor and adds the highest-CIS candidate. After ~10 iterations, the anchor set stabilizes at high quality -- achieving with 100 anchors what baselines need 500+ for.

## Step-by-Step Workflow

1. **Construct the performance matrix.** Collect binary evaluation results into a matrix `S` of shape `(num_models, num_items)` where `S[i,j] = 1` if model `i` answered item `j` correctly and `-1` otherwise. Source data from the Open LLM Leaderboard, your own evaluation runs, or any benchmark with per-item results.

2. **Split models into train/validation/test.** Randomly partition the model axis (rows) -- e.g., 60% train, 20% validation, 20% test. Training models are used to learn anchor weights; validation models select hyperparameters and initialization strategy; test models evaluate final performance.

3. **Discover sparsity structure.** Compute the item-item cosine similarity matrix from `S_train`. Apply spectral clustering (e.g., `sklearn.cluster.SpectralClustering` with 5 clusters) to verify that items group into coherent blocks. Visualize the similarity matrix sorted by cluster to confirm diagonal block structure.

4. **Initialize anchors via k-means.** Run k-means clustering on the item vectors (columns of `S_train`) with `k = num_anchors` (start with 100). Select the item closest to each cluster centroid as the initial anchor. Also try random initialization on the validation set and pick whichever yields lower MAE.

5. **Train the MLP weight optimizer.** Build a small MLP (2-3 hidden layers, ReLU activations) that takes the anchor-masked performance row as input and outputs the predicted full-benchmark score. Train with Adam optimizer, MSE loss, for E epochs (e.g., 100-200). The MLP learns nonlinear weight combinations of anchor items.

6. **Compute residuals and importance scores.** After training, compute model-level residuals `e = predicted_scores - true_scores`. For each anchor `i`, compute `AIS_i = ||dL/dS(:,i)||_1` via backpropagation. For each non-anchor candidate `j`, compute `CIS_j = |S(:,j)^T * e|`.

7. **Iteratively refine the anchor set.** For R iterations (default 10): remove the anchor with the lowest AIS, add the candidate with the highest CIS, retrain the MLP, and recompute scores. Track validation MAE and Kendall's tau at each step to monitor convergence.

8. **Evaluate on held-out test models.** Using the final anchor set and trained MLP, predict full-benchmark scores for test models using only their anchor-item results. Report MAE (target: < 2%) and Kendall's tau (target: > 0.9) against true scores.

9. **Export the anchor set.** Save the list of selected anchor item indices/IDs and the trained MLP weights. These are reusable: any new model only needs to be evaluated on the anchor items to get a reliable full-benchmark score estimate.

10. **Deploy for cheap evaluation.** Integrate the anchor set into your evaluation pipeline. Run new models only on the anchor subset, feed results through the trained MLP, and obtain predicted rankings without full benchmark inference.

## Concrete Examples

**Example 1: Reducing MMLU evaluation cost**

User: "I have evaluation results from 200 models on the full MMLU benchmark (14,042 items). I want to find a subset of ~100 items that can predict MMLU scores for new models."

Approach:
1. Load the 200x14042 binary performance matrix from CSV
2. Split models: 120 train, 40 validation, 40 test
3. Run spectral clustering on item vectors to verify sparsity (expect 5 coherent clusters)
4. Initialize 100 anchors via k-means on item vectors
5. Train a 2-layer MLP (input: 100, hidden: 64, output: 1) for 150 epochs
6. Run 10 refinement iterations swapping low-AIS anchors for high-CIS candidates
7. Evaluate: MAE ~0.84%, Kendall's tau ~0.91 on test models

Output:
```python
import torch
import numpy as np
from sklearn.cluster import KMeans, SpectralClustering

# Load performance matrix: rows=models, cols=items, values in {-1, 1}
S = np.loadtxt("mmlu_results.csv", delimiter=",")  # shape (200, 14042)

# Train/val/test split on model axis
idx = np.random.permutation(200)
S_train, S_val, S_test = S[idx[:120]], S[idx[120:160]], S[idx[160:]]

# Step 1: Verify sparsity via spectral clustering
sim = np.corrcoef(S_train.T)  # item-item similarity
clustering = SpectralClustering(n_clusters=5, affinity='precomputed')
labels = clustering.fit_predict((sim + 1) / 2)  # normalize to [0,1]

# Step 2: Initialize anchors via k-means
k = 100
kmeans = KMeans(n_clusters=k).fit(S_train.T)
anchors = []
for c in range(k):
    members = np.where(kmeans.labels_ == c)[0]
    center = kmeans.cluster_centers_[c]
    closest = members[np.argmin(np.linalg.norm(S_train.T[members] - center, axis=1))]
    anchors.append(closest)
anchors = np.array(anchors)

# Step 3: Train MLP
true_scores = S_train.mean(axis=1)  # full benchmark accuracy per model
anchor_inputs = S_train[:, anchors]  # only anchor columns

model = torch.nn.Sequential(
    torch.nn.Linear(k, 64), torch.nn.ReLU(),
    torch.nn.Linear(64, 32), torch.nn.ReLU(),
    torch.nn.Linear(32, 1)
)
optimizer = torch.optim.Adam(model.parameters(), lr=1e-3)
X = torch.tensor(anchor_inputs, dtype=torch.float32)
y = torch.tensor(true_scores, dtype=torch.float32).unsqueeze(1)

for epoch in range(150):
    pred = model(X)
    loss = torch.nn.functional.mse_loss(pred, y)
    optimizer.zero_grad()
    loss.backward()
    optimizer.step()

# Step 4: Refinement loop (simplified)
candidates = set(range(S_train.shape[1])) - set(anchors)
for r in range(10):
    # Compute AIS for each anchor
    ais = []
    for i, a in enumerate(anchors):
        col = torch.tensor(S_train[:, a], dtype=torch.float32, requires_grad=True)
        # ... backprop through trained model to get gradient norm
        ais.append(gradient_l1_norm)
    worst = anchors[np.argmin(ais)]

    # Compute CIS for each candidate
    residuals = model(X).detach().numpy().flatten() - true_scores
    cis = {c: abs(S_train[:, c] @ residuals) for c in candidates}
    best_candidate = max(cis, key=cis.get)

    # Swap
    anchors = np.array([a for a in anchors if a != worst] + [best_candidate])
    candidates = candidates - {best_candidate} | {worst}
    # Retrain MLP on updated anchors...

# Final: save anchor indices for deployment
np.save("mmlu_anchors_100.npy", anchors)
torch.save(model.state_dict(), "mmlu_mlp_weights.pt")
```

**Example 2: Comparing 50 fine-tuned model variants cheaply**

User: "I fine-tuned 50 variants of Llama on different data mixes. I already ran GSM8K fully on 10 of them. Can I rank all 50 without running the full 1,319 items on every variant?"

Approach:
1. Use the 10 fully-evaluated models to build the initial performance matrix (10x1319)
2. Augment with Open LLM Leaderboard data for GSM8K (thousands more models) to get a richer training signal
3. Select 100 anchors from the combined matrix using k-means + refinement
4. Run only the 100 anchor items on the remaining 40 model variants
5. Predict full GSM8K scores via the trained MLP
6. Report predicted rankings with confidence intervals

Output:
```
Model Variant       | Predicted GSM8K (%) | Estimated Rank
--------------------|--------------------|--------------
llama-datamix-17    | 72.4               | 1
llama-datamix-03    | 71.8               | 2
llama-datamix-42    | 70.1               | 3
...                 | ...                | ...
llama-datamix-29    | 45.2               | 50

Evaluation cost: 100 items x 40 models = 4,000 inferences
Full cost would be: 1,319 items x 40 models = 52,760 inferences
Reduction: 92.4%
Expected ranking accuracy: Kendall's tau > 0.93
```

**Example 3: Building a compact CI evaluation suite**

User: "I want a fast smoke test for our model training pipeline. We currently run 6 benchmarks (ARC, GSM8K, HellaSwag, MMLU, TruthfulQA, Winogrande). Can we get a single compact test set?"

Approach:
1. Download per-item results from the Open LLM Leaderboard for all 6 benchmarks
2. Run SparseEval independently on each benchmark to select 50 anchors per benchmark
3. Combine into a 300-item evaluation suite (vs ~25,000+ total items across all 6)
4. Validate on held-out models: predict per-benchmark scores and overall ranking
5. Export as a JSONL file with item IDs and the 6 trained MLP weight files

Output:
```
Compact CI Suite Summary:
  ARC:        50 anchors (of 1,172) | MAE 1.2% | tau 0.92
  GSM8K:      50 anchors (of 1,319) | MAE 1.6% | tau 0.94
  HellaSwag:  50 anchors (of 10,042)| MAE 0.8% | tau 0.92
  MMLU:       50 anchors (of 14,042)| MAE 0.9% | tau 0.91
  TruthfulQA: 50 anchors (of 817)  | MAE 1.0% | tau 0.93
  Winogrande: 50 anchors (of 1,267)| MAE 1.0% | tau 0.90

Total items: 300 (down from ~28,659)
Estimated CI runtime reduction: ~99x
```

## Best Practices

- **Do:** Use binary performance matrices ({-1, 1} or {0, 1}). The sparsity structure depends on discrete correctness signals, not continuous scores. Convert probability outputs to binary via thresholding first.
- **Do:** Include as many models as possible when building the performance matrix. SparseEval leverages cross-model patterns -- 200+ models gives much better anchor selection than 20. Supplement with Open LLM Leaderboard data when your own model count is small.
- **Do:** Always compare k-means and random initialization on a validation set. The paper shows k-means usually wins, but not always -- dataset structure matters.
- **Do:** Start with 100 anchors as a baseline, then experiment with fewer (50, 25) to find the cost-accuracy tradeoff that suits your use case. At 100 anchors, expect MAE < 2% and tau > 0.9 on standard benchmarks.
- **Avoid:** Skipping the iterative refinement step. AIS/CIS-based swapping consistently improves results across all benchmarks, especially when anchor budgets are tight.
- **Avoid:** Using a linear model instead of an MLP for weight optimization. The paper's ablation shows the MLP's nonlinearity is critical -- linear models plateau at higher error rates. Use at least 2 hidden layers.
- **Avoid:** Applying anchors trained on one benchmark to a different benchmark. Anchor sets are task-specific; the sparsity structure differs across ARC, MMLU, GSM8K, etc. Train separate anchor sets per benchmark.

## Error Handling

- **Too few models in training set:** If you have fewer than 50 models with full evaluation results, anchor quality degrades. Mitigate by augmenting with public leaderboard data from Hugging Face Open LLM Leaderboard.
- **No clear cluster structure:** If spectral clustering on the item similarity matrix shows no diagonal blocks, the benchmark may lack redundancy. In this case, SparseEval will still work but with higher MAE -- increase the anchor count or accept reduced compression.
- **MLP overfitting:** With very few training models, the MLP may overfit. Use early stopping on validation loss, reduce hidden layer sizes, or add dropout (0.1-0.2). Monitor the gap between train and validation MAE.
- **Anchor refinement diverges:** If MAE increases during refinement iterations, the CIS signal is noisy. Reduce the refinement rate (swap fewer anchors per iteration) or increase training epochs between swaps.
- **New model falls outside training distribution:** If a new model is architecturally very different from training models (e.g., mixture-of-experts vs dense transformers), predicted scores may be unreliable. Validate periodically by fully evaluating a new model and checking prediction error.

## Limitations

- **Requires historical evaluation data.** You need a performance matrix from many models evaluated on the full benchmark before you can select anchors. This is a one-time cost, but it means SparseEval cannot help with a brand-new benchmark that has no prior evaluations.
- **Task-specific anchors.** Anchor sets do not transfer across benchmarks. Each evaluation task needs its own anchor selection process.
- **Binary correctness assumption.** The method is designed for right/wrong evaluation items. For benchmarks with continuous scores (e.g., BLEU, ROUGE), you would need to binarize or adapt the formulation.
- **Rank preservation, not exact scores.** While MAE is low, the primary strength is ranking models correctly (high Kendall's tau). If you need precise per-model accuracy numbers rather than rankings, use larger anchor sets or verify with periodic full evaluations.
- **Cold-start for model families.** If the training models are all from one family (e.g., all Llama variants), anchors may not generalize well to structurally different model families (e.g., Mamba or mixture-of-experts).

## Reference

- **Paper:** [SparseEval: Efficient Evaluation of Large Language Models by Sparse Optimization](https://arxiv.org/abs/2602.07909v1) (ICLR 2026). Focus on Section 3 (method) for the MLP optimization formulation and AIS/CIS scoring, and Section 4 (experiments) for anchor-count ablations showing the 5x reduction over baselines.
- **Code:** [https://github.com/taolinzhang/SparseEval](https://github.com/taolinzhang/SparseEval) -- run `bash SparseEval/run/gd_cluster_mlp.sh <dataset> <num_anchors>` for the full pipeline.