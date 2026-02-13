---
name: "trust-one-round-confidence"
description: "Estimate LLM output confidence from structural signals in a single pass -- no multi-sampling needed. Use when: 'check if this answer is reliable', 'score confidence of this response', 'flag hallucinations in this output', 'how trustworthy is this generation', 'detect uncertain claims', 'add confidence scores to LLM pipeline'."
---

# Structural Confidence: Single-Pass LLM Output Reliability Scoring

This skill enables Claude to design and implement confidence estimation systems for LLM outputs using the Structural Confidence framework from Yang et al. (2026). Instead of requiring multiple expensive sampling rounds or relying on brittle token probabilities, this approach extracts spectral, local-variation, and shape-coherence signals from hidden-state trajectories of a single forward pass through a proxy encoder. The result is a 70-dimensional structural descriptor that a lightweight classifier maps to a correctness probability -- achieving competitive accuracy with 5-6x lower compute than sampling-based alternatives like SelfCheckGPT.

## When to Use

- When the user wants to build a confidence scoring layer for an LLM-powered pipeline (RAG, QA, fact-checking)
- When the user asks how to detect hallucinations without running the model multiple times
- When implementing a verification gate that decides whether to trust, escalate, or regenerate an LLM response
- When the user needs a model-agnostic confidence signal that works across different LLMs without internal access
- When building a production system where multi-sample consistency checks are too slow or expensive
- When the user asks to rank or triage LLM answers by reliability before presenting them to users

## Key Technique

**The core insight:** Confident LLM outputs produce smooth, low-frequency hidden-state trajectories with compact geometry, while uncertain or hallucinated outputs exhibit spectral irregularity, sharp local fluctuations, and dispersed global structure. This pattern holds even when measured through a frozen proxy encoder (BERT-base) rather than the generating model itself, because the stability signals are induced by the structure of the context-answer text, not by model-specific parameters.

**The framework** concatenates the context (question or prompt) with the LLM's answer, passes this through a frozen BERT-base encoder capped at 256 tokens, and extracts three families of features from the resulting hidden-state matrix H: (1) **Spectral stability** -- 48 features from the real-valued DFT (lowest 16 non-trivial frequencies) and the 16 smallest eigenvalues of the normalized graph Laplacian; (2) **Local variation** -- 6 features including total path length, mean/variance of consecutive L2 displacements, start-end distance, embedding-wise variance, and centroid norm; (3) **Shape coherence** -- a 16-bin normalized histogram of all pairwise distances between hidden states. These concatenate into a 70-dimensional descriptor u(H).

**Two-scale aggregation** improves robustness: features are computed once over the full trajectory (global) and once over overlapping windows of width 5 with stride 2 (local), then element-wise averaged. A LightGBM classifier (200 trees, learning rate 0.05, binary logistic loss) maps the descriptor to a correctness probability. Optionally, fusing with a sentence embedding (e.g., text-embedding-3-large) further boosts discrimination on some domains.

## Step-by-Step Workflow

1. **Prepare the input pair.** Concatenate the original prompt/context with the LLM's generated answer into a single string. Truncate or chunk to stay within 256 tokens for the proxy encoder.

2. **Encode through the proxy model.** Pass the concatenated text through a frozen BERT-base (or equivalent transformer encoder). Extract the final-layer hidden states, yielding a matrix H of shape (T, d) where T is the token count and d is the hidden dimension (768 for BERT-base).

3. **Compute spectral stability features (48-dim).** Apply a real-valued DFT to each hidden dimension across the token axis; retain the lowest 16 non-trivial frequency magnitudes (32 features). Build a token-adjacency graph, compute the normalized Laplacian, and extract the 16 smallest eigenvalues (16 features).

4. **Compute local variation features (6-dim).** Calculate L2 norms between consecutive hidden states h_i and h_{i+1}. Aggregate into: total path length (sum of norms), mean displacement, variance of displacement, start-to-end distance ||h_1 - h_T||, embedding-wise variance across all positions, and centroid norm ||mean(H)||.

5. **Compute shape coherence features (16-dim).** Calculate all pairwise Euclidean distances ||h_i - h_j|| for i < j. Build a 16-bin normalized histogram of these distances to capture the global geometry of the trajectory.

6. **Apply two-scale aggregation.** Repeat steps 3-5 using overlapping windows (width=5, stride=2) across the token sequence, averaging the per-window features to get local descriptors. Element-wise average the global and local descriptors to produce the final 70-dimensional vector u(H).

7. **Score with the classifier.** Feed u(H) into a trained LightGBM model to obtain a correctness probability in [0, 1]. If using the fused variant, concatenate a sentence-level embedding of the answer before classification.

8. **Threshold and act.** Apply a decision threshold (calibrate on held-out data): scores above threshold are trusted, scores below trigger regeneration, human review, or a fallback response. Use Brier score and expected calibration error to tune the threshold.

9. **Log and monitor.** Store the structural descriptor alongside each prediction for drift detection. Track the distribution of confidence scores over time to catch systematic shifts in model behavior.

## Concrete Examples

**Example 1: Adding confidence gating to a RAG pipeline**

User: "I have a RAG system that retrieves documents and generates answers with GPT-4. Some answers hallucinate facts not in the retrieved context. Build me a confidence gate that flags unreliable answers without calling the model again."

Approach:
1. Install dependencies: `transformers` (BERT-base), `numpy`, `scipy`, `lightgbm`.
2. Create a `StructuralConfidence` class that:
   - Loads `bert-base-uncased` frozen (no gradient computation).
   - Accepts `(context, answer)` pairs.
   - Extracts the 70-dim descriptor using spectral, local, and shape features.
3. Collect a labeled calibration set: ~500 (context, answer, correct/incorrect) triples from your domain.
4. Train a LightGBM classifier on the descriptors.
5. In the RAG pipeline, after generation, run the confidence gate. If score < 0.5, return "I'm not confident in this answer" or trigger re-retrieval.

Output structure:
```python
from structural_confidence import StructuralConfidence

gate = StructuralConfidence(
    encoder="bert-base-uncased",
    classifier_path="models/confidence_lgbm.pkl",
    threshold=0.5
)

answer = rag_pipeline.generate(query, documents)
score = gate.score(context=documents, answer=answer)

if score < gate.threshold:
    answer = fallback_strategy(query, documents)
    answer.metadata["confidence"] = "low"
else:
    answer.metadata["confidence"] = f"{score:.2f}"
```

**Example 2: Batch-scoring LLM outputs for quality triage**

User: "I generated 10,000 QA pairs with an LLM for a dataset. I need to rank them by reliability so human reviewers check the worst ones first."

Approach:
1. Build the structural descriptor extractor as a batch pipeline.
2. For each (question, answer) pair, compute the 70-dim vector.
3. If no labeled data is available for your domain, use the cross-domain pretrained classifier (trained on FEVER + SciFact + WikiBio mix -- 5,455 instances).
4. Sort by ascending confidence score. Present the bottom quartile to reviewers first.

Output:
```python
import pandas as pd
from structural_confidence import StructuralConfidence

gate = StructuralConfidence(encoder="bert-base-uncased",
                            classifier_path="models/cross_domain_lgbm.pkl")

df = pd.read_parquet("qa_pairs.parquet")
df["confidence"] = df.apply(
    lambda r: gate.score(context=r["question"], answer=r["answer"]),
    axis=1
)
df_sorted = df.sort_values("confidence")
df_sorted.to_parquet("qa_pairs_scored.parquet")

# Bottom 25% flagged for human review
flagged = df_sorted.head(len(df_sorted) // 4)
print(f"Flagged {len(flagged)} pairs for review (confidence < {flagged['confidence'].max():.3f})")
```

**Example 3: Implementing the feature extraction from scratch**

User: "Show me how to extract the structural features from hidden states."

Output:
```python
import numpy as np
from scipy.sparse.csgraph import laplacian
from scipy.sparse import csr_matrix

def spectral_features(H: np.ndarray, n_freq: int = 16, n_eigen: int = 16) -> np.ndarray:
    """48-dim spectral stability descriptor."""
    T, d = H.shape
    # DFT: lowest n_freq non-trivial frequencies across hidden dims
    fft_coeffs = np.fft.rfft(H, axis=0)[1:n_freq+1]  # skip DC
    dft_feats = fft_coeffs.real.mean(axis=1)  # (n_freq,) -- real part
    dft_imag = fft_coeffs.imag.mean(axis=1)   # (n_freq,) -- imag part
    # Graph Laplacian: adjacency from consecutive tokens
    row = np.arange(T - 1)
    col = np.arange(1, T)
    weights = np.array([np.linalg.norm(H[i] - H[i+1]) for i in range(T - 1)])
    adj = csr_matrix((weights, (row, col)), shape=(T, T))
    adj = adj + adj.T
    lap = laplacian(adj, normed=True).toarray()
    eigenvalues = np.sort(np.linalg.eigvalsh(lap))[:n_eigen]
    return np.concatenate([dft_feats, dft_imag, eigenvalues])

def local_variation_features(H: np.ndarray) -> np.ndarray:
    """6-dim local variation descriptor."""
    displacements = np.linalg.norm(np.diff(H, axis=0), axis=1)
    path_length = displacements.sum()
    mean_disp = displacements.mean()
    var_disp = displacements.var()
    start_end = np.linalg.norm(H[0] - H[-1])
    embed_var = H.var(axis=0).mean()
    centroid_norm = np.linalg.norm(H.mean(axis=0))
    return np.array([path_length, mean_disp, var_disp, start_end, embed_var, centroid_norm])

def shape_coherence_features(H: np.ndarray, n_bins: int = 16) -> np.ndarray:
    """16-dim shape coherence descriptor."""
    from scipy.spatial.distance import pdist
    dists = pdist(H, metric="euclidean")
    hist, _ = np.histogram(dists, bins=n_bins, density=True)
    return hist

def structural_descriptor(H: np.ndarray, window: int = 5, stride: int = 2) -> np.ndarray:
    """70-dim two-scale structural descriptor."""
    def extract(h):
        return np.concatenate([
            spectral_features(h),
            local_variation_features(h),
            shape_coherence_features(h)
        ])
    global_desc = extract(H)
    # Local: overlapping windows
    T = H.shape[0]
    local_descs = []
    for start in range(0, T - window + 1, stride):
        local_descs.append(extract(H[start:start + window]))
    local_desc = np.mean(local_descs, axis=0) if local_descs else global_desc
    return (global_desc + local_desc) / 2.0
```

## Best Practices

- **Do:** Use BERT-base as the proxy encoder -- the paper validates that proxy-derived signals transfer well and you avoid needing access to the generating model's internals.
- **Do:** Cap input at 256 tokens. The pairwise distance computation is O(T^2), and longer sequences add cost without proportional benefit.
- **Do:** Train on domain-specific labeled data when available. Cross-domain transfer works but domain-specific classifiers outperform by 5-15% AUROC on in-distribution data.
- **Do:** Use the two-scale (global + local) aggregation. It consistently outperforms single-scale variants across all benchmarks.
- **Avoid:** Using structural confidence as the sole arbiter for safety-critical decisions. It is a probabilistic signal, not a guarantee. Combine with other checks.
- **Avoid:** Skipping calibration. Raw LightGBM probabilities need calibration (Platt scaling or isotonic regression) to be interpreted as true confidence values.

## Error Handling

| Problem | Cause | Mitigation |
|---------|-------|------------|
| Very short answers (< 5 tokens) | Window-based local features degenerate | Fall back to global-only descriptors; flag as low-information |
| Sequence exceeds 256 tokens | BERT truncation loses tail content | Chunk the input, compute descriptors per chunk, average |
| Spectral features return NaN | Token count < n_freq + 1 for DFT | Pad the frequency vector with zeros or reduce n_freq |
| Laplacian eigendecomposition is slow | Large T or dense adjacency | Already mitigated by 256-token cap; use sparse eigensolver |
| Classifier outputs extreme probabilities (0.0 or 1.0) | Uncalibrated LightGBM | Apply Platt scaling on a held-out validation set |
| Cross-domain performance drops | Domain shift between training and deployment data | Fine-tune the LightGBM on a small labeled set from the target domain (50-100 examples often suffice) |

## Limitations

- **No internal model access required, but quality depends on proxy alignment.** If the generating LLM's failure modes differ sharply from what BERT-base can detect in text structure, the structural signals may miss certain error types.
- **Binary correctness only.** The framework scores correct vs. incorrect, not degree of correctness. Partially correct answers are hard to calibrate.
- **Requires labeled calibration data.** Even the cross-domain variant needs a labeled training pool. Zero-shot structural confidence is not supported.
- **Factual errors with fluent structure can evade detection.** A grammatically smooth, well-structured hallucination may produce a calm hidden-state trajectory, yielding a false high-confidence score.
- **Evaluated primarily on fact-verification and QA tasks.** Performance on code generation, mathematical reasoning, or creative writing is unvalidated.

## Reference

Yang, P., Wen, J., Jin, H., Huang, L., & Chen, H. (2026). *Trust in One Round: Confidence Estimation for Large Language Models via Structural Signals.* arXiv:2602.00977v1. [https://arxiv.org/abs/2602.00977v1](https://arxiv.org/abs/2602.00977v1)

Key sections to read: Section 3 (Structural Descriptors) for the feature extraction math, Section 4 (Experiments) for benchmark-specific tuning, and Table 2 for the cross-domain transfer results that inform how to deploy without domain-specific training data.