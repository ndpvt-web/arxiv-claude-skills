---
name: "kv-core-benchmarking-data-dependent-low-rank"
description: "Benchmark and analyze KV-cache low-rank compressibility in LLMs using SVD-based evaluation and Normalized Effective Rank. Use when: 'analyze KV-cache compressibility', 'profile which layers can be compressed', 'benchmark KV-cache rank per layer', 'compute effective rank of attention caches', 'optimize KV-cache memory with SVD', 'layer-wise compression analysis for transformers'."
---

# KV-CoRE: Benchmarking Data-Dependent Low-Rank Compressibility of KV-Caches

This skill enables Claude to implement and apply the KV-CoRE framework for quantifying how compressible each layer's key-value cache is in a transformer model. The core technique uses Singular Value Decomposition (SVD) to compute the optimal low-rank approximation of KV matrices under the Frobenius norm, then summarizes compressibility via the Normalized Effective Rank (NER) metric. Because the method is gradient-free and supports incremental updates, it can profile entire datasets layer-by-layer without backpropagation or full recomputation, producing actionable per-layer compression budgets.

## When to Use

- When the user wants to **profile which transformer layers** have KV-caches that can be aggressively compressed without quality loss.
- When building a **dynamic KV-cache compression system** that allocates different ranks to different layers based on data characteristics.
- When the user asks to **benchmark memory savings** achievable by low-rank KV-cache approximation on a specific model and dataset.
- When comparing **GQA vs MHA architectures** in terms of cache compressibility.
- When investigating how **domain or language** (e.g., code vs. natural language vs. math) affects the intrinsic dimensionality of attention caches.
- When the user needs to select a **per-layer rank budget** for deploying a KV-cache compression scheme in production inference.

## Key Technique

**SVD-based compressibility measurement.** For each transformer layer, collect the Key matrix K and Value matrix V accumulated during inference over a dataset. Each matrix has shape `(num_tokens, head_dim * num_heads)` (or `(num_tokens, head_dim * num_kv_heads)` for GQA). Apply SVD: `K = U_k @ diag(sigma_k) @ V_k^T`. The singular values `sigma` capture how much variance each rank-1 component explains. A rapidly decaying spectrum means the matrix is well-approximated at low rank; a flat spectrum means it is not.

**Normalized Effective Rank (NER).** Rather than picking an arbitrary variance threshold, NER provides a single scalar summarizing the spectral shape. Given singular values `sigma_1, ..., sigma_n`, compute: `NER = (sum(sigma_i))^2 / (n * sum(sigma_i^2))`. This value lies in `[1/n, 1]`. Lower NER means energy is concentrated in fewer singular values, indicating higher compressibility. The paper shows NER correlates strongly with actual task performance degradation under compression, making it a reliable proxy metric that avoids expensive downstream evaluation.

**Incremental computation.** When processing a dataset token-by-token or batch-by-batch, rather than re-running full SVD on the growing KV matrix, the method uses incremental SVD updates. When a new batch of rows arrives, update the existing decomposition in O(r^2 * d) time instead of O(n * d^2), where r is the current rank approximation, d is the head dimension, and n is the sequence length. This makes dataset-scale profiling practical.

## Step-by-Step Workflow

1. **Select the model and dataset.** Load the target LLM (e.g., Llama-3, Mistral, Qwen) and a representative dataset for the deployment domain. Use at least 500-1000 samples to get stable NER estimates.

2. **Instrument the forward pass to capture KV-caches.** Register hooks on each transformer layer's attention module to intercept the K and V tensors after projection but before any existing compression. Store them in CPU memory or accumulate their SVD incrementally.

3. **Accumulate KV matrices per layer.** For each input sample, concatenate the K and V matrices across the sequence dimension. Maintain separate accumulators for keys and values at each layer. Shape per layer: `(total_tokens_so_far, head_dim * num_kv_heads)`.

4. **Compute SVD per layer.** For each layer's accumulated K and V matrices, run truncated SVD (e.g., `torch.linalg.svd` with `full_matrices=False`, or `scipy.sparse.linalg.svds` for large matrices). Extract the singular values vector `sigma`.

5. **Calculate Normalized Effective Rank.** For each layer and each of K/V:
   ```python
   def normalized_effective_rank(sigma):
       sigma = sigma[sigma > 1e-10]  # drop numerical zeros
       n = len(sigma)
       ner = (sigma.sum() ** 2) / (n * (sigma ** 2).sum())
       return ner.item()
   ```

6. **Build a layer-wise compressibility profile.** Create a table mapping `(layer_index, key_or_value) -> NER`. Sort layers by NER to identify the most and least compressible layers. Visualize as a bar chart or heatmap.

7. **Determine per-layer rank budgets.** For each layer, compute the rank `r` needed to retain a target fraction of the Frobenius norm: `cumsum(sigma^2) / sum(sigma^2) >= target` (e.g., target = 0.95). Layers with low NER will need far fewer ranks than layers with high NER.

8. **Estimate memory savings.** For each layer, compute the compression ratio: `original_memory = seq_len * d` vs `compressed_memory = seq_len * r + r * d` (storing the low-rank factors). Sum across layers to get total savings.

9. **Validate with downstream task evaluation.** Apply the computed rank budgets to the actual KV-cache during inference (reconstruct from low-rank factors). Measure perplexity or task accuracy to confirm NER predictions hold.

10. **Iterate on thresholds.** If degradation exceeds tolerance, increase the variance retention target for the offending layers. If memory budget allows, increase ranks uniformly. The NER profile guides where additional ranks help most.

## Concrete Examples

**Example 1: Profile a Llama-3-8B model on English text**

User: "I want to know which layers of Llama-3-8B have the most compressible KV-caches when running on Wikipedia text."

Approach:
1. Load Llama-3-8B and a 1000-sample Wikipedia subset.
2. Run inference with KV-cache hooks capturing K and V per layer (32 layers).
3. Compute SVD and NER for all 64 matrices (32 keys + 32 values).
4. Output a ranked table.

Output:
```
Layer  | Key NER | Value NER | Key Rank@95% | Value Rank@95% | Compressible?
-------|---------|-----------|--------------|----------------|-------------
  0    |  0.71   |   0.68    |     412      |     398        | Low
  1    |  0.65   |   0.61    |     384      |     356        | Low
  ...
  15   |  0.38   |   0.34    |     198      |     172        | Medium
  ...
  28   |  0.19   |   0.16    |      87      |      64        | High
  29   |  0.17   |   0.14    |      72      |      58        | High
  30   |  0.21   |   0.18    |      94      |      76        | High
  31   |  0.23   |   0.20    |     102      |      84        | High

Summary: Layers 24-31 show NER < 0.25, suitable for 60%+ rank reduction.
         Layers 0-5 show NER > 0.6, compress conservatively (< 20% reduction).
         Estimated total KV-cache memory reduction: 38% at 95% variance retention.
```

**Example 2: Compare compressibility across domains**

User: "Compare KV-cache compressibility on code vs. natural language vs. math for Mistral-7B."

Approach:
1. Load Mistral-7B and three datasets: The Stack (code), C4 (natural language), GSM8K (math).
2. Run KV-CoRE profiling on each dataset (500 samples each).
3. Compute mean NER per layer for each domain.
4. Present a comparative analysis.

Output:
```python
import matplotlib.pyplot as plt

# Results (mean NER across all layers)
domains = {"Code": 0.28, "Natural Language": 0.34, "Math": 0.41}

# Per-layer NER for layer 16 (middle):
#   Code: 0.22, NL: 0.29, Math: 0.37

# Interpretation:
# - Code has most repetitive attention patterns -> highest compressibility
# - Math requires more diverse attention -> lowest compressibility
# - Deploy different rank budgets per domain for optimal memory/quality tradeoff
```

**Example 3: Implement incremental SVD for streaming evaluation**

User: "I need to compute KV-cache compressibility without storing the full matrix in memory."

Approach:
1. Use incremental SVD to process batches of tokens.
2. Maintain a running low-rank approximation per layer.
3. Update NER after each batch.

Output:
```python
import torch
import numpy as np

class IncrementalSVDProfiler:
    def __init__(self, max_rank=256):
        self.max_rank = max_rank
        self.U = None       # Left singular vectors
        self.sigma = None   # Singular values
        self.Vt = None      # Right singular vectors (transposed)

    def update(self, new_rows: torch.Tensor):
        """Incrementally update SVD with new KV-cache rows.
        new_rows: shape (batch_tokens, head_dim * num_kv_heads)
        """
        new_rows = new_rows.float().cpu().numpy()

        if self.U is None:
            U, s, Vt = np.linalg.svd(new_rows, full_matrices=False)
            r = min(self.max_rank, len(s))
            self.U, self.sigma, self.Vt = U[:, :r], s[:r], Vt[:r, :]
            return

        # Form the augmented matrix: [diag(sigma) @ Vt; new_rows]
        top = np.diag(self.sigma) @ self.Vt
        augmented = np.vstack([top, new_rows])

        U_aug, s_aug, Vt_aug = np.linalg.svd(augmented, full_matrices=False)
        r = min(self.max_rank, len(s_aug))
        self.sigma = s_aug[:r]
        self.Vt = Vt_aug[:r, :]
        # U is not needed for NER; skip storing it to save memory

    def normalized_effective_rank(self) -> float:
        s = self.sigma[self.sigma > 1e-10]
        n = len(s)
        if n == 0:
            return 1.0
        return float((s.sum() ** 2) / (n * (s ** 2).sum()))

    def rank_at_variance(self, target=0.95) -> int:
        cumvar = np.cumsum(self.sigma ** 2)
        total = cumvar[-1]
        return int(np.searchsorted(cumvar, target * total) + 1)
```

## Best Practices

- **Do:** Profile on representative data from the actual deployment domain. KV-cache compressibility is data-dependent -- the whole point of this method. A profile from Wikipedia will not predict behavior on code or mathematical reasoning.
- **Do:** Always profile keys and values separately. They often have very different spectral profiles within the same layer. Values tend to be more compressible than keys in most architectures.
- **Do:** Use at least 500 samples for stable NER estimates. Single-sample measurements are noisy and not representative of dataset-level behavior.
- **Do:** Account for GQA when interpreting results. GQA models (Llama-3, Mistral) have fewer KV heads, so their matrices are already lower-dimensional, often yielding 15-25% better compressibility than equivalent MHA models.
- **Avoid:** Applying a single uniform compression ratio across all layers. The entire insight of KV-CoRE is that layers vary dramatically -- early layers are typically 2-3x less compressible than later layers.
- **Avoid:** Using NER alone without downstream validation for production deployments. NER is a strong proxy metric, but always verify with perplexity or task accuracy on a held-out set before committing to a rank budget.

## Error Handling

- **Out-of-memory during SVD:** If the accumulated KV matrix is too large for full SVD, switch to incremental SVD (Example 3) or use randomized SVD (`sklearn.utils.extmath.randomized_svd`) which operates in O(n * d * r) memory.
- **Numerically zero singular values:** Filter `sigma > 1e-10` before computing NER to avoid division instability. This occurs naturally when sequence length exceeds the embedding dimension.
- **Inconsistent NER across runs:** Ensure the same tokenization and batching order. NER is deterministic given the same input data, but different token orderings in batches can shift incremental SVD results slightly. Average over 2-3 runs if precision matters.
- **GQA head dimension mismatch:** For GQA models, reshape KV-caches to `(seq_len, num_kv_heads * head_dim)` not `(seq_len, num_attention_heads * head_dim)`. Using the wrong head count inflates the matrix with repeated values and corrupts NER.

## Limitations

- **Static profiling, not runtime compression.** KV-CoRE tells you *how compressible* each layer is -- it does not itself perform inference-time compression. You need a separate compression implementation (e.g., apply truncated SVD to KV-caches during decoding) to realize memory savings.
- **Cost of profiling.** Full SVD on large KV matrices is O(n * d^2). For very long contexts (128K+ tokens), even incremental SVD can be slow. Consider subsampling tokens or using randomized SVD for initial estimates.
- **NER is a Frobenius-norm proxy.** It measures approximation quality in Frobenius norm, which may not perfectly predict attention-weighted reconstruction quality. Attention-aware metrics could refine predictions but are more expensive to compute.
- **No guidance on mixed-precision interaction.** The method assumes full-precision KV-caches. The interaction between quantization (e.g., FP8/INT4 KV-caches) and low-rank approximation is not characterized.
- **Language coverage gaps.** The benchmark covers 16 languages but low-resource languages may have unreliable NER estimates due to limited training data in the model, not due to inherent language properties.

## Reference

- **Paper:** [KV-CoRE: Benchmarking Data-Dependent Low-Rank Compressibility of KV-Caches in LLMs](https://arxiv.org/abs/2602.05929v2) (Chen et al., 2026). Look for: the NER formula definition (Section 3), the incremental SVD update procedure (Section 3.2), the layer-wise compressibility heatmaps (Section 5), and the NER-vs-performance-degradation correlation analysis (Section 6).