---
name: "modality-gap-driven-subspace-alignment"
description: "Align multimodal embeddings (vision-language) by correcting the modality gap using the ReAlign/ReVision technique. Fixes geometric misalignment between image and text embeddings from CLIP-like encoders via mean-shift, trace-scaling, and centroid correction — without retraining. Use when: 'align CLIP embeddings', 'fix modality gap', 'bridge vision-language representations', 'train MLLM with unpaired text data', 'improve cross-modal retrieval accuracy', 'reduce hallucination in multimodal LLM'."
---

# Modality Gap-Driven Subspace Alignment (ReAlign / ReVision)

This skill enables Claude to apply the ReAlign algorithm — a training-free, statistics-driven method for correcting the geometric modality gap between vision and language embeddings from dual-encoder models (CLIP, SigLIP, etc.). Instead of assuming the gap is a simple uniform offset (isotropic), ReAlign decomposes it into a principal bias, an orthogonal bias, and anisotropic residuals, then corrects each component through three linear operations: Anchor, Trace, and Centroid Alignment. The technique also powers ReVision, a two-stage MLLM training paradigm that uses ReAlign-transformed unpaired text as a substitute for expensive image-text pairs during pretraining.

## When to Use

- When a user wants to improve cross-modal retrieval (image-to-text or text-to-image) by closing the modality gap in CLIP/SigLIP embeddings
- When building or fine-tuning a Multimodal LLM and the user wants to reduce dependence on large-scale paired image-text datasets
- When a user reports that their text and image embeddings cluster in separate regions of the embedding space and wants to fix this without retraining encoders
- When implementing a zero-shot classification or retrieval pipeline and accuracy degrades due to systematic offset between modalities
- When a user wants to create pseudo-visual embeddings from text-only data for pretraining an MLLM vision adapter
- When debugging hallucination in multimodal models that may stem from misaligned representation distributions

## Key Technique

**The Modality Gap Problem.** Dual-encoder models like CLIP project images and text onto a shared unit hypersphere, but embeddings of the same semantic content occupy systematically different regions. This gap is not a simple translation — it has anisotropic structure. Prior corrections (like C3) assume isotropic offsets, which destroys the spectral hierarchy of the embedding covariance and degrades downstream performance.

**Fixed-Frame Modality Gap Theory.** The gap `Delta = e_image - e_text` is decomposed within a frozen reference frame built from the top-r eigenvectors of the pooled covariance `Cov(e_image) + Cov(e_text)`. This yields four terms: (1) **Principal Modality Bias** — the mean gap projected onto the signal subspace U, (2) **Constant Orthogonal Bias** — the mean gap in the null subspace V, (3) **U-residuals** — zero-mean, highly anisotropic fluctuations in U (condition number > 10^3), and (4) **V-residuals** — zero-mean stretched fluctuations orthogonal to U. The key insight: the bias terms are stable and correctable, while the residuals carry task-relevant anisotropic structure that must be preserved.

**ReAlign corrects biases without destroying anisotropy** through three steps that use only first- and second-order statistics from unpaired data (no paired images and text required). These statistics stabilize with as few as 10,000 samples per modality.

## Step-by-Step Workflow

### Applying ReAlign (Training-Free Embedding Alignment)

1. **Collect unpaired embedding statistics.** Encode a large sample of images (N >= 10,000) through the vision encoder to get `{e_x}`, and a separate large sample of text through the text encoder to get `{e_y}`. These do NOT need to be paired. Compute: mean `mu_x = mean(e_x)`, mean `mu_y = mean(e_y)`, trace `T_x = mean(||e_x - mu_x||^2)`, trace `T_y = mean(||e_y - mu_y||^2)`.

2. **Anchor Alignment (mean shift).** For each text embedding, subtract the text mean and add the image mean: `e_y_anchored = (e_y - mu_y) + mu_x`. This eliminates the first-order distributional shift — the largest component of the modality gap.

3. **Trace Alignment (variance scaling).** Compute the scaling factor `s = sqrt(T_x / (T_y + epsilon))`. Apply: `e_y_traced = mu_x + s * (e_y - mu_y)`. This matches the global variance of text embeddings to that of image embeddings while preserving the covariance structure and spectral hierarchy. Use `epsilon = 1e-8` for numerical stability.

4. **L2-normalize the traced embeddings.** Compute `e_y_normed = e_y_traced / ||e_y_traced||`. This projects back onto the unit hypersphere. Note: this normalization introduces a "Phantom Drift" — a spurious centroid shift caused by nonlinear projection of anisotropic distributions.

5. **Centroid Alignment (correct Phantom Drift).** Compute the post-normalization mean `mu_normed = mean(e_y_normed)` over your text sample. Correct: `e_y_final = e_y_normed - mu_normed + mu_x_normed`, then re-normalize: `e_y_final = e_y_final / ||e_y_final||`. Here `mu_x_normed` is the mean of L2-normalized image embeddings.

6. **Validate alignment.** Measure the modality gap as the L2 distance between the centroids of aligned text and image embeddings. A well-aligned result should show gap < 1e-3 (vs. typical pre-alignment gaps of ~0.05-0.10). Also verify that k-NN mixing rate (fraction of cross-modal neighbors in top-k) improves.

### Applying ReVision (MLLM Training Paradigm)

7. **Stage 1 — Modality Substitution Pretraining.** Apply ReAlign to map text embeddings into the image embedding distribution, producing pseudo-visual tokens. Train the vision-language adapter (e.g., an MLP projector) on these pseudo-visual tokens paired with the original text, using next-token prediction loss. Keep the LLM backbone frozen. This requires only unpaired text data (~1M samples).

8. **Stage 2 — Visual Instruction Tuning.** Fine-tune the adapter (and optionally the LLM) on real image-instruction pairs using standard supervised fine-tuning. Because Stage 1 pre-aligned the adapter to the visual distribution, this stage needs fewer paired samples for the same performance.

9. **Inference.** At inference time, use real image embeddings directly (no ReAlign needed). The asymmetric design — aligning text-to-image during training — means the adapter already expects image-distribution inputs.

10. **Monitor spectral preservation.** After alignment, check that the eigenvalue spectrum of the aligned text covariance retains a power-law decay with exponent alpha >= 1.3. If alpha drops below ~1.1, the alignment may be over-smoothing — reduce the sample size or verify statistics.

## Concrete Examples

**Example 1: Fixing Cross-Modal Retrieval with CLIP**

User: "My CLIP text-to-image retrieval has poor recall. Text and image embeddings seem to cluster separately. How do I fix the modality gap?"

Approach:
1. Encode 50,000 random images from the dataset through CLIP's vision encoder
2. Encode 50,000 random text captions (can be from a different source) through CLIP's text encoder
3. Compute statistics: `mu_x`, `mu_y`, `T_x`, `T_y`
4. Apply the three ReAlign steps to all query text embeddings at retrieval time
5. Use aligned text embeddings for cosine similarity search against image embeddings

```python
import numpy as np

# Precompute statistics (do once)
mu_x = image_embeds.mean(axis=0)
mu_y = text_embeds.mean(axis=0)
T_x = np.mean(np.sum((image_embeds - mu_x) ** 2, axis=1))
T_y = np.mean(np.sum((text_embeds - mu_y) ** 2, axis=1))

def realign(e_y, mu_x, mu_y, T_x, T_y, mu_x_normed, eps=1e-8):
    # Step 1: Anchor
    centered = e_y - mu_y
    # Step 2: Trace
    s = np.sqrt(T_x / (T_y + eps))
    traced = mu_x + s * centered
    # Step 3: Normalize + Centroid correction
    normed = traced / (np.linalg.norm(traced, axis=-1, keepdims=True) + eps)
    # Compute post-norm mean over a batch or precomputed
    mu_normed = normed.mean(axis=0)
    corrected = normed - mu_normed + mu_x_normed
    return corrected / (np.linalg.norm(corrected, axis=-1, keepdims=True) + eps)

# Precompute mu_x_normed
image_normed = image_embeds / np.linalg.norm(image_embeds, axis=-1, keepdims=True)
mu_x_normed = image_normed.mean(axis=0)

# At query time
aligned_queries = realign(query_text_embeds, mu_x, mu_y, T_x, T_y, mu_x_normed)
scores = aligned_queries @ image_embeds.T  # cosine similarity
```

**Example 2: Pretraining an MLLM Vision Adapter with Unpaired Text**

User: "I want to pretrain a vision-language adapter for LLaVA-style model but I don't have enough image-text pairs. Can I use text-only data?"

Approach:
1. Compute ReAlign statistics from 50K images (any images) and 50K texts (any texts) encoded through the vision/text encoders
2. Encode 1M text samples through the text encoder
3. Apply ReAlign to produce pseudo-visual embeddings
4. Train the MLP projector to map pseudo-visual tokens to the LLM's input space using next-token prediction on the original text

```python
import torch
import torch.nn as nn

class ReAlignTransform:
    def __init__(self, mu_x, mu_y, T_x, T_y, mu_x_normed):
        self.mu_x = torch.tensor(mu_x, dtype=torch.float32)
        self.mu_y = torch.tensor(mu_y, dtype=torch.float32)
        self.scale = (T_x / (T_y + 1e-8)) ** 0.5
        self.mu_x_normed = torch.tensor(mu_x_normed, dtype=torch.float32)
        self._mu_post_norm = None  # set after first pass

    def __call__(self, e_y):
        centered = e_y - self.mu_y
        traced = self.mu_x + self.scale * centered
        normed = torch.nn.functional.normalize(traced, dim=-1)
        if self._mu_post_norm is not None:
            corrected = normed - self._mu_post_norm + self.mu_x_normed
            return torch.nn.functional.normalize(corrected, dim=-1)
        return normed

    def calibrate_centroid(self, text_loader, text_encoder):
        """Run once on a large text batch to estimate post-norm mean."""
        all_normed = []
        for batch in text_loader:
            with torch.no_grad():
                e_y = text_encoder(batch)
                traced = self.mu_x + self.scale * (e_y - self.mu_y)
                normed = torch.nn.functional.normalize(traced, dim=-1)
                all_normed.append(normed)
        self._mu_post_norm = torch.cat(all_normed).mean(dim=0)

# Stage 1: Modality Substitution Pretraining
realign = ReAlignTransform(mu_x, mu_y, T_x, T_y, mu_x_normed)
realign.calibrate_centroid(text_loader, text_encoder)

for text_batch in training_loader:
    text_embeds = text_encoder(text_batch)        # [B, D]
    pseudo_visual = realign(text_embeds)           # [B, D] in image space
    projected = mlp_projector(pseudo_visual)       # [B, LLM_dim]
    loss = next_token_loss(llm, projected, text_batch)
    loss.backward()
    optimizer.step()
```

**Example 3: Diagnosing Modality Gap Severity**

User: "How do I measure whether my multimodal embeddings have a significant modality gap?"

Approach:
1. Encode matched image-text pairs through both encoders
2. Compute the four-component decomposition to quantify each source of misalignment
3. Report actionable metrics

```python
# Diagnostic script
image_embs = encode_images(images)  # [N, D]
text_embs = encode_texts(texts)     # [N, D]

# Overall gap
gap = (image_embs.mean(0) - text_embs.mean(0))
gap_norm = np.linalg.norm(gap)
print(f"Centroid gap (L2): {gap_norm:.4f}")  # > 0.01 = significant

# Construct reference frame (top-r eigenvectors of pooled covariance)
pooled_cov = np.cov(image_embs.T) + np.cov(text_embs.T)
eigenvalues, eigenvectors = np.linalg.eigh(pooled_cov)
idx = eigenvalues.argsort()[::-1]
r = np.sum(eigenvalues[idx] > eigenvalues[idx[0]] * 0.01)  # effective rank
U = eigenvectors[:, idx[:r]]  # signal subspace

# Decompose
pmb = U @ (U.T @ gap)         # Principal Modality Bias
cob = gap - pmb               # Constant Orthogonal Bias
print(f"PMB norm: {np.linalg.norm(pmb):.4f}")
print(f"COB norm: {np.linalg.norm(cob):.4f}")

# Anisotropy check
deltas = image_embs - text_embs  # [N, D]
delta_U = deltas @ U @ U.T
cov_U = np.cov(delta_U.T)
eig_U = np.linalg.eigvalsh(cov_U)
kappa = eig_U[-1] / (eig_U[0] + 1e-10)
print(f"U-residual condition number: {kappa:.0f}")  # > 100 = highly anisotropic

# k-NN mixing rate
from sklearn.neighbors import NearestNeighbors
all_embs = np.concatenate([image_embs, text_embs])
labels = np.array([0]*len(image_embs) + [1]*len(text_embs))
nn = NearestNeighbors(n_neighbors=10).fit(all_embs)
_, indices = nn.kneighbors(all_embs)
mixing = np.mean(labels[indices[:, 1:]] != labels[:, None])
print(f"k-NN mixing rate: {mixing:.2%}")  # < 5% = severe gap
```

## Best Practices

- **Do:** Compute statistics from at least 10,000 samples per modality. Alignment quality degrades sharply below this threshold.
- **Do:** Use unpaired data for statistics — the whole point of ReAlign is that paired data is not required for the alignment transform.
- **Do:** Always apply the Centroid Alignment (Step 3) after normalization. Skipping it leaves a Phantom Drift artifact that worsens with embedding anisotropy.
- **Do:** Verify spectral preservation by checking that the power-law exponent of the aligned covariance eigenvalues stays above ~1.3.
- **Avoid:** Using isotropic whitening (ZCA, PCA-whitening) to close the gap — this flattens the eigenvalue spectrum and destroys the anisotropic structure that carries semantic information.
- **Avoid:** Applying ReAlign at inference time in an MLLM pipeline. The ReVision design is asymmetric: align text-to-image during training so that real images work natively at inference.

## Error Handling

- **Statistics from too few samples:** If N < 10,000, the trace estimates `T_x` and `T_y` will be noisy. The scaling factor `s` may overshoot, causing aligned embeddings to have incorrect variance. Solution: increase sample size or use bootstrap confidence intervals to verify stability.
- **Degenerate trace ratio:** If `T_y` is near zero (text embeddings are nearly constant), the scale factor explodes. Guard with `epsilon = 1e-8` and log a warning if `s > 10`.
- **Post-normalization mean is inaccurate:** The Centroid Alignment step requires `mu_normed` computed over a large batch. If computed from a small batch, the Phantom Drift correction will be incomplete. Use at least 5,000 samples for this estimate.
- **Encoder mismatch:** ReAlign statistics are specific to the encoder checkpoint. If you swap or fine-tune the encoder, you must recompute all statistics.
- **Dimension mismatch between encoders:** ReAlign assumes both encoders produce embeddings in the same dimensionality. If they differ, apply a learned linear projection first.

## Limitations

- ReAlign is a **post-hoc correction for frozen encoders**. It cannot fix gaps caused by fundamentally different semantic coverage between modalities (e.g., if the vision encoder lacks concept coverage the text encoder has).
- The technique assumes embeddings live on or near a unit hypersphere (L2-normalized). For non-normalized embedding spaces, the Trace and Centroid steps need adaptation.
- ReVision's Stage 1 (unpaired pretraining) improves data efficiency but does **not eliminate the need for paired data entirely** — Stage 2 still requires image-instruction pairs for visual grounding.
- The Fixed-Frame Theory assumes the reference frame is stable across training. For models undergoing aggressive fine-tuning of the encoders, the decomposition may not hold.
- Performance gains are most pronounced when the original modality gap is significant (centroid distance > 0.01). For already well-aligned models, ReAlign provides diminishing returns.

## Reference

**Paper:** Yu et al., "Modality Gap-Driven Subspace Alignment Training Paradigm For Multimodal Large Language Models" (2026). [arXiv:2602.07026](https://arxiv.org/abs/2602.07026v1)

**What to look for:** Section 3 for the Fixed-Frame Modality Gap Theory and four-component decomposition; Section 4 for the ReAlign algorithm (Anchor/Trace/Centroid steps with proofs); Section 5 for ReVision two-stage training; Appendix E.1 for sample-size ablation showing N=10K stability threshold.