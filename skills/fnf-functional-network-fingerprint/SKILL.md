---
name: "fnf-functional-network-fingerprint"
description: "Detect whether a suspect LLM is derived from a victim model using Functional Network Fingerprints (FNF). Applies neuroscience-inspired ICA decomposition to intermediate layer activations to build training-free, modification-robust model fingerprints. Triggers: 'fingerprint this model', 'check if model is derived from', 'LLM intellectual property detection', 'model provenance verification', 'detect model theft', 'compare model lineage'."
---

# Functional Network Fingerprint (FNF) for LLM Provenance Detection

This skill enables Claude to implement and apply the Functional Network Fingerprint method -- a training-free, sample-efficient technique for determining whether a suspect large language model was derived from a known victim model. FNF works by extracting intermediate-layer activations from both models on a small shared input set, decomposing those activations into independent functional networks via Canonical Independent Component Analysis (CanICA), and comparing the resulting fingerprints using cosine similarity. The method is robust to fine-tuning, pruning, quantization, and even cross-architecture comparison, making it a practical tool for LLM intellectual property protection.

## When to Use

- When a user wants to check whether a fine-tuned, pruned, or quantized model was derived from a specific base model (e.g., "Was this model fine-tuned from LLaMA-2?")
- When building an automated IP verification pipeline that must detect unauthorized model derivatives
- When comparing two models of different sizes or architectures to determine if they share a common training origin
- When a user asks to implement model fingerprinting or provenance tracking for an LLM registry
- When verifying that a released model checkpoint matches the claimed base model lineage
- When a user needs a non-invasive model comparison technique that does not require retraining or watermark injection

## Key Technique

FNF treats an LLM as a digital brain and borrows functional neuroimaging analysis methods to identify its internal "functional networks." During inference on a small set of input samples (as few as 10-50), forward hooks capture the hidden state outputs from each transformer layer's MLP block. These activation matrices (shape: `[num_samples, hidden_dim]`) are then decomposed using CanICA -- a variant of Independent Component Analysis that first applies randomized SVD for dimensionality reduction, then runs FastICA with multiple random seeds to find the sparsest decomposition (minimizing the L-infinity norm). The resulting independent components represent functional networks: groups of neurons that fire together consistently.

The core insight is that models sharing a common origin -- even after substantial fine-tuning, structured pruning, or weight permutation -- preserve highly consistent functional network patterns. The mixing matrices from ICA decomposition are z-score normalized, thresholded (values below the 99th percentile of absolute values are zeroed), and polarity-corrected to ensure dominant signs are positive. This produces a sparse binary-like fingerprint vector per model. Cosine similarity between fingerprint vectors from two models yields a score near 1.0 for derived models and significantly lower (typically below 0.5) for independently trained models. This gap is reliable enough for automated provenance detection.

Unlike watermarking or weight-hashing approaches, FNF requires no modification to the model during training, works across different architectures and hidden dimensions (by comparing ICA components in a shared low-dimensional space), and needs only a handful of inference passes on arbitrary text samples.

## Step-by-Step Workflow

1. **Install dependencies.** Ensure `torch`, `transformers`, `scikit-learn`, `numpy`, and `scipy` are available. Clone the reference implementation from `https://github.com/WhatAboutMyStar/LLM_ACTIVATION` if using the `llmfact` library directly.

2. **Load both models.** Load the victim (base) model and the suspect model using HuggingFace Transformers with `device_map="auto"`. Both models must be loadable for inference but do not need to share the same architecture or hidden dimension.

3. **Prepare a shared input set.** Select 10-50 diverse text samples (e.g., from Wikipedia, C4, or any general-purpose corpus). Tokenize them with each model's own tokenizer. The same semantic content must be fed to both models, but tokenization can differ.

4. **Register forward hooks on transformer layers.** For each model, register `forward_hook` callbacks on every `model.layers.{i}` (or equivalent decoder layer) to capture the hidden state output tensor. Store outputs on CPU to manage GPU memory:

   ```python
   activations = {}
   def make_hook(name):
       def hook(module, input, output):
           # output may be a tuple; extract the hidden state tensor
           if isinstance(output, tuple):
               output = output[0]
           activations[name] = output.detach().cpu()
       return hook

   for i, layer in enumerate(model.model.layers):
       layer.register_forward_hook(make_hook(f"layer.{i}"))
   ```

5. **Run inference and collect activation matrices.** Pass each input sample through the model under `torch.no_grad()`. After each forward pass, concatenate the per-layer hidden states into a single activation matrix of shape `[num_samples, num_layers * hidden_dim]`. Alternatively, analyze per-layer activations independently for finer-grained comparison.

6. **Apply CanICA decomposition.** For each model's activation matrix:
   - Apply randomized SVD to reduce to `n_components` dimensions (default: 20).
   - Run FastICA with 10 different random seeds; select the result with the smallest L-infinity norm (sparsest solution).
   - Z-score normalize each component: `(component - mean) / std`.
   - Threshold: zero out values below the 99th percentile of absolute values.
   - Polarity-correct: flip component signs so the dominant direction is positive.

   ```python
   from sklearn.utils.extmath import randomized_svd
   from sklearn.decomposition import FastICA
   import numpy as np

   def compute_fingerprint(activation_matrix, n_components=20, n_seeds=10):
       U, S, Vt = randomized_svd(activation_matrix, n_components=n_components)
       reduced = U * S  # Project to low-dim space

       best_mixing, best_sparsity = None, float("inf")
       for seed in range(n_seeds):
           ica = FastICA(n_components=n_components, random_state=seed)
           ica.fit(reduced)
           mixing = ica.mixing_
           sparsity = np.max(np.sum(np.abs(mixing), axis=0))
           if sparsity < best_sparsity:
               best_mixing, best_sparsity = mixing, sparsity

       # Z-score normalize
       normed = (best_mixing - best_mixing.mean(axis=0)) / (best_mixing.std(axis=0) + 1e-8)
       # Threshold at 99th percentile
       thresh = np.percentile(np.abs(normed), 99)
       normed[np.abs(normed) < thresh] = 0
       # Polarity correction
       for col in range(normed.shape[1]):
           if np.sum(normed[:, col]) < 0:
               normed[:, col] *= -1
       return normed.flatten()
   ```

7. **Compute cosine similarity between fingerprints.** Compare the flattened fingerprint vectors from the victim and suspect models:

   ```python
   from numpy.linalg import norm
   similarity = np.dot(fp_victim, fp_suspect) / (norm(fp_victim) * norm(fp_suspect) + 1e-8)
   ```

8. **Apply decision threshold.** A cosine similarity above ~0.7 strongly indicates shared origin. Scores above 0.85 are near-certain matches. Scores below 0.4 indicate independent training. The gray zone (0.4-0.7) warrants additional investigation with more samples or per-layer analysis.

9. **Optional: per-layer analysis.** For higher confidence, compute fingerprints independently for each transformer layer and report a layer-wise similarity profile. Derived models typically show high similarity across all layers, while coincidental matches show inconsistent layer-wise scores.

10. **Report results.** Output the overall similarity score, per-layer breakdown if computed, and a determination (derived / not derived / inconclusive) with the evidence.

## Concrete Examples

**Example 1: Checking if a fine-tuned chat model derives from LLaMA-2-7B**

User: "I found a model on HuggingFace called 'SuperChat-7B' that claims to be trained from scratch, but it looks suspiciously similar to LLaMA-2-7B. Can you write code to verify its provenance?"

Approach:
1. Load `meta-llama/Llama-2-7b-hf` as the victim model and `someone/SuperChat-7B` as the suspect.
2. Prepare 30 Wikipedia text samples as shared inputs.
3. Extract activations from all 32 transformer layers of each model.
4. Run CanICA with `n_components=20` on each model's concatenated activation matrix.
5. Compute cosine similarity between the two fingerprints.

Output:
```
Victim model:  meta-llama/Llama-2-7b-hf (32 layers, 4096 hidden dim)
Suspect model: someone/SuperChat-7B (32 layers, 4096 hidden dim)
Input samples: 30 (Wikipedia)
ICA components: 20

Overall fingerprint cosine similarity: 0.923

Per-layer similarity (top-5 most discriminative):
  Layer 0:  0.97
  Layer 15: 0.94
  Layer 24: 0.91
  Layer 31: 0.89
  Layer 8:  0.88

Determination: DERIVED (high confidence)
The suspect model's functional network activity is highly consistent with the
victim model, indicating shared origin despite any post-training modifications.
```

**Example 2: Ruling out derivation between independently trained models**

User: "Compare Mistral-7B and Qwen-7B to confirm they are independently developed."

Approach:
1. Load both models. Note they have different tokenizers and architectures.
2. Use the same 50 text samples, tokenized independently with each model's tokenizer.
3. Extract per-layer activations, applying CanICA decomposition to each.
4. Since hidden dimensions may differ, compare in the ICA component space (both reduced to `n_components=20`).

Output:
```
Model A: mistralai/Mistral-7B-v0.1 (32 layers, 4096 hidden dim)
Model B: Qwen/Qwen-7B (32 layers, 4096 hidden dim)
Input samples: 50 (Wikipedia)
ICA components: 20

Overall fingerprint cosine similarity: 0.18

Determination: NOT DERIVED (high confidence)
The models exhibit fundamentally different functional network organizations,
consistent with independent training on different data/objectives.
```

**Example 3: Detecting derivation after aggressive pruning**

User: "Someone released a pruned 4-bit quantized version of our model. Can FNF still detect the lineage?"

Approach:
1. Load the original full-precision model and the pruned/quantized suspect.
2. Extract activations from both (quantized model will produce lower-precision activations but the ICA decomposition normalizes this).
3. Run the standard fingerprint pipeline.
4. Expect slightly reduced but still significant similarity due to functional network preservation.

Output:
```
Victim model:  our-org/OriginalModel-13B (fp16)
Suspect model: anon/PrunedModel-13B-4bit (int4, 50% sparsity)
Input samples: 20 (C4 dataset)
ICA components: 20

Overall fingerprint cosine similarity: 0.81

Determination: DERIVED (moderate-high confidence)
Despite aggressive pruning and 4-bit quantization, core functional networks
remain intact. Similarity is reduced compared to full-precision derivatives
but remains well above the independent-training baseline.
```

## Best Practices

- **Do:** Use diverse, general-purpose text samples (Wikipedia, C4, news) rather than domain-specific data. Functional networks are best characterized by broad activation patterns, not narrow task-specific ones.
- **Do:** Run CanICA with multiple random seeds (at least 10) and select the sparsest decomposition. This makes the fingerprint stable and reproducible across runs.
- **Do:** Normalize and threshold the mixing matrices before comparison. Raw ICA outputs are scale-variant and noisy; z-score normalization with 99th-percentile thresholding removes noise while preserving the structural signal.
- **Do:** Report per-layer similarity alongside the aggregate score. This provides interpretable evidence and catches edge cases where only certain layers were modified.
- **Avoid:** Using fewer than 10 input samples. Below this threshold, the activation matrix lacks sufficient statistical diversity for reliable ICA decomposition.
- **Avoid:** Comparing fingerprints computed with different `n_components` values. Both models must use the same number of ICA components for the cosine similarity to be meaningful.

## Error Handling

- **OOM during activation collection:** Large models may exhaust GPU memory when storing all layer activations simultaneously. Mitigate by processing layers in batches (e.g., 8 layers at a time), moving activations to CPU immediately, or using gradient checkpointing mode.
- **ICA convergence failure:** FastICA may fail to converge on certain seeds. The multi-seed approach handles this gracefully -- skip failed seeds and select the best among successful runs. Require at least 3 successful seeds.
- **Mismatched architectures:** When comparing models with different hidden dimensions, you cannot directly concatenate layer activations into the same matrix. Instead, run CanICA independently on each model and compare in the shared `n_components`-dimensional ICA space.
- **Tokenizer differences:** Different tokenizers produce different sequence lengths for the same text. This is expected and handled naturally since activations are averaged or concatenated across the sequence dimension before ICA.
- **Near-threshold scores (0.4-0.7):** These are inconclusive. Increase the number of input samples to 100+, try per-layer analysis, or use samples from multiple domains to improve discrimination.

## Limitations

- FNF requires inference access to both models. It cannot work with API-only models that do not expose intermediate layer activations.
- Very small models (under ~100M parameters) may not develop sufficiently distinct functional networks for reliable fingerprinting.
- The method has been validated primarily on decoder-only transformer architectures (LLaMA, Mistral, Qwen families). Encoder-only or encoder-decoder models may require adaptation of the layer selection strategy.
- Models that share a common pre-training dataset but were trained independently from different random initializations could show moderate similarity due to data-driven convergence, potentially producing false positives in the gray zone.
- The `n_components` hyperparameter affects sensitivity: too few components miss fine-grained differences; too many introduce noise. The default of 20 is empirically validated for 7B-13B scale models but may need adjustment for significantly larger or smaller models.

## Reference

**Paper:** Liu et al., "FNF: Functional Network Fingerprint for Large Language Models" (arXiv:2601.22692v1, 2026). Look for Section 3 (Method) for the CanICA decomposition pipeline and Section 4 (Experiments) for robustness evaluations across fine-tuning, pruning, and cross-architecture settings.

**Code:** https://github.com/WhatAboutMyStar/LLM_ACTIVATION -- see `LLM-Fingerprint.ipynb` for the end-to-end fingerprinting workflow and `llmfact/decomposition/ica.py` for the CanICA implementation.