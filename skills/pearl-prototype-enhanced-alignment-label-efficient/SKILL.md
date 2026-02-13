---
name: "pearl-prototype-enhanced-alignment-label-efficient"
description: |
  Implements PEARL (Prototype-Enhanced Aligned Representation Learning) to improve embedding quality for nearest-neighbor retrieval, similarity search, and lightweight classifiers when labeled data is scarce. Reshapes embedding geometry by softly aligning vectors toward class prototypes without retraining the base encoder.
  Trigger phrases:
  - "improve my embedding quality for retrieval"
  - "fix nearest neighbor search accuracy with few labels"
  - "align embeddings toward class prototypes"
  - "post-process embeddings for better similarity search"
  - "label-efficient embedding refinement"
  - "improve kNN classification with limited labels"
---

# PEARL: Prototype-Enhanced Alignment for Label-Efficient Embedding Refinement

This skill enables Claude to implement the PEARL method for improving pre-computed embedding quality in retrieval and classification systems where labeled data is scarce. Instead of retraining an expensive base encoder, PEARL trains a lightweight refinement network that reshapes embedding geometry by aligning vectors toward normalized class prototypes. The method preserves original dimensionality and avoids aggressive projection or collapse, making it a practical drop-in improvement for deployed systems using sentence encoders (e.g., OpenAI, Cohere, BGE, or Sentence-Transformers).

## When to Use

- When the user has pre-computed embeddings from a frozen encoder and nearest-neighbor retrieval returns wrong classes
- When building a similarity search or case-routing system with only 100-5000 labeled examples
- When the user wants to improve kNN classifier accuracy without retraining the embedding model
- When embedding neighborhoods are noisy — similar vectors belong to different classes
- When deploying a text classification or retrieval system on a new domain where labels are expensive to obtain
- When unsupervised post-processing (PCA whitening, centering) gives inconsistent gains and the user wants a supervised alternative
- When the user needs to improve a citizen message routing, support ticket triage, or document retrieval pipeline

## Key Technique

**The core problem:** Pre-trained embeddings produce high-dimensional vectors where local neighborhoods (the nearest neighbors of any point) often contain vectors from the wrong class. This directly degrades kNN classifiers, similarity-based routing, and retrieval systems. Unsupervised fixes like PCA whitening help inconsistently. Fully supervised projections like LDA require thousands of labels.

**PEARL's approach:** Given a small labeled subset, compute a normalized prototype (centroid) for each class by averaging and L2-normalizing the labeled embeddings per class. Then train a lightweight encoder-decoder network with a split architecture: a *signal encoder* that captures class-discriminative structure, and a *residual encoder* that preserves remaining information. The signal encoder is trained with six combined objectives: (1) reconstruction loss to preserve information, (2) full reconstruction from signal+residual, (3) cosine alignment toward the class prototype, (4) contrastive loss with temperature τ=0.1 to push apart different-class representations, (5) cross-entropy classification for stability, and (6) an orthogonality penalty to keep signal and residual components separated. The final refined embedding is the signal encoder's output, L2-normalized for cosine retrieval.

**Why it works:** The multi-objective design gently reshapes local neighborhood geometry so that same-class items cluster tighter while different-class items separate — without collapsing the embedding space into a low-dimensional projection. At 100 labels, PEARL achieves 25.7% improvement in neighborhood purity over raw embeddings and 21.1% over unsupervised post-processing baselines.

## Step-by-Step Workflow

1. **Collect and embed your corpus.** Use your existing encoder (Sentence-Transformers, OpenAI `text-embedding-3-small`, BGE, etc.) to produce embeddings for all documents. Store as a NumPy array of shape `(N, d)` where `d` is the embedding dimension.

2. **Gather a small labeled subset.** You need as few as 100 labeled examples spanning your target classes. More labels (300-1200) yield progressively better results, but gains plateau beyond ~2500. Ensure every class has at least a few representatives.

3. **Compute class prototypes.** For each class `c`, average all labeled embeddings belonging to that class, then L2-normalize the result:
   ```python
   import numpy as np

   def compute_prototypes(embeddings, labels):
       classes = np.unique(labels)
       prototypes = {}
       for c in classes:
           mask = labels == c
           centroid = embeddings[mask].mean(axis=0)
           prototypes[c] = centroid / np.linalg.norm(centroid)
       return prototypes
   ```

4. **Build the PEARL refinement network.** Construct two encoders — signal `E_s` mapping to `d_s` dimensions, and residual `E_r` mapping to `d_r` dimensions — plus decoders for reconstruction, a projection head for prototype alignment, and a lightweight classifier head:
   ```python
   import torch
   import torch.nn as nn

   class PEARLRefiner(nn.Module):
       def __init__(self, d_in, d_signal=256, d_residual=128, n_classes=10):
           super().__init__()
           self.signal_enc = nn.Sequential(
               nn.Linear(d_in, 512), nn.ReLU(), nn.Linear(512, d_signal))
           self.residual_enc = nn.Sequential(
               nn.Linear(d_in, 256), nn.ReLU(), nn.Linear(256, d_residual))
           self.decoder_signal = nn.Sequential(
               nn.Linear(d_signal, 512), nn.ReLU(), nn.Linear(512, d_in))
           self.decoder_full = nn.Sequential(
               nn.Linear(d_signal + d_residual, 512), nn.ReLU(), nn.Linear(512, d_in))
           self.proj = nn.Linear(d_signal, d_in)  # project back for prototype alignment
           self.classifier = nn.Linear(d_signal, n_classes)

       def forward(self, x):
           z_s = self.signal_enc(x)
           z_r = self.residual_enc(x)
           return z_s, z_r
   ```

5. **Implement the six-objective loss function.** Combine reconstruction, alignment, contrastive, classification, and orthogonality losses:
   ```python
   def pearl_loss(model, x, labels, prototypes, tau=0.1, w_full=0.5):
       z_s, z_r = model(x)
       # Reconstruction losses
       x_hat_s = model.decoder_signal(z_s)
       x_hat_full = model.decoder_full(torch.cat([z_s, z_r], dim=-1))
       L_recon = ((x_hat_s - x) ** 2).mean()
       L_full = w_full * ((x_hat_full - x) ** 2).mean()
       # Prototype alignment (cosine)
       proj_z = model.proj(z_s)
       proj_z = proj_z / proj_z.norm(dim=-1, keepdim=True)
       target_protos = torch.stack([prototypes[y.item()] for y in labels])
       L_align = (1 - (proj_z * target_protos).sum(dim=-1)).mean()
       # Contrastive loss over prototypes
       all_protos = torch.stack(list(prototypes.values()))  # (C, d)
       sims = (proj_z @ all_protos.T) / tau
       proto_labels = torch.tensor([list(prototypes.keys()).index(y.item()) for y in labels])
       L_contrast = nn.functional.cross_entropy(sims, proto_labels)
       # Classification
       logits = model.classifier(z_s)
       L_cls = nn.functional.cross_entropy(logits, proto_labels)
       # Orthogonality
       z_s_norm = z_s / z_s.norm(dim=-1, keepdim=True)
       z_r_norm = z_r / z_r.norm(dim=-1, keepdim=True)
       L_orth = (z_s_norm * z_r_norm).sum(dim=-1).abs().mean()
       return L_recon + L_full + L_align + L_contrast + L_cls + L_orth
   ```

6. **Train the refiner on labeled data only.** Use standard mini-batch SGD or Adam for 50-200 epochs on the labeled subset. The network is small, so training is fast (seconds to minutes).

7. **Apply the trained signal encoder to all embeddings.** Pass your full corpus through the signal encoder, then L2-normalize:
   ```python
   with torch.no_grad():
       z_s, _ = model(torch.tensor(all_embeddings))
       refined = z_s / z_s.norm(dim=-1, keepdim=True)
   ```

8. **Evaluate neighborhood quality.** Measure Purity@K (fraction of K nearest neighbors sharing the query label), Hit@K (whether at least one neighbor is correct), and MRR@K on a held-out set. Compare against raw embeddings and any baselines.

9. **Deploy the refined embeddings.** Replace your original embeddings in the vector store (FAISS, Pinecone, Qdrant, pgvector) with the refined versions. For new incoming documents, run them through the base encoder then the PEARL signal encoder before indexing or querying.

10. **Monitor and retrain incrementally.** As new labeled data arrives, recompute prototypes with the expanded label set and retrain the lightweight refiner. This is cheap — no base encoder retraining needed.

## Concrete Examples

**Example 1: Improving support ticket routing with few labels**

User: "I have 50,000 support tickets embedded with `text-embedding-3-small`. I've manually labeled 200 tickets across 15 categories. My kNN router misclassifies about 40% of tickets. Can you help improve the routing accuracy?"

Approach:
1. Load the 50K embeddings (shape `50000 x 1536`) and the 200 labeled examples
2. Compute 15 class prototypes by averaging and normalizing the labeled embeddings per category
3. Build a PEARL refiner with `d_in=1536, d_signal=384, d_residual=192, n_classes=15`
4. Train for 100 epochs on the 200 labeled examples with combined loss
5. Apply the signal encoder to all 50K embeddings, L2-normalize
6. Re-index into the vector store and evaluate

Output:
```
Prototypes computed for 15 classes (min 8, max 22 examples per class)
Training PEARL refiner: epoch 100/100, loss=0.43
Purity@5 improved: 0.58 → 0.74 (+27.6%)
kNN@5 F1 improved: 0.61 → 0.78 (+27.9%)
Refined embeddings saved to refined_embeddings.npy (50000 x 384)
```

**Example 2: Enhancing document retrieval in a legal corpus**

User: "Our legal search system uses BGE-large embeddings. Retrieval quality is poor for some case types. We have 500 labeled case-type annotations across 30 categories. How can we improve retrieval without retraining BGE?"

Approach:
1. Load existing BGE-large embeddings (dim=1024) for the full corpus
2. Split the 500 labels into 400 train / 100 validation
3. Compute 30 class prototypes from the 400 training labels
4. Build PEARL refiner: `d_in=1024, d_signal=512, d_residual=256, n_classes=30`
5. Train with early stopping on validation Purity@5
6. Evaluate Hit@10 and MRR@10 on held-out queries

Output:
```
Validation Purity@5: 0.42 (raw) → 0.56 (PEARL) [+33.3%]
Hit@10: 0.71 → 0.84
MRR@10: 0.39 → 0.52
Retrained in 45 seconds on CPU. Ready for deployment.
```

**Example 3: Using the pearl-H package directly**

User: "I want to quickly apply PEARL to my embeddings. Is there a ready-made package?"

Approach:
1. Install the package: `pip install pearl-H`
2. Use it with your embeddings and labels:

```python
from pearl_h import PEARL

# embeddings: np.ndarray of shape (N, d)
# labels: np.ndarray of shape (N_labeled,) — indices into embeddings
# label_values: class labels for the labeled subset

model = PEARL(n_classes=15, d_signal=256)
model.fit(embeddings[labeled_indices], label_values, epochs=100)
refined = model.transform(embeddings)
# refined is (N, d_signal), L2-normalized, ready for kNN or vector search
```

## Best Practices

- **Do:** Start with a signal dimension `d_signal` between 1/4 and 1/2 of the original embedding dimension. This compresses while retaining discriminative information.
- **Do:** Ensure every class has at least 3-5 labeled examples. Classes with only 1 example produce unstable prototypes.
- **Do:** L2-normalize both prototypes and refined embeddings before cosine similarity operations. PEARL is designed for cosine-space retrieval.
- **Do:** Use stratified splits for validation to ensure all classes are represented when evaluating neighborhood quality.
- **Avoid:** Setting the contrastive temperature τ too low (< 0.05), which causes gradient instability, or too high (> 0.5), which weakens the contrastive signal. Start with τ=0.1.
- **Avoid:** Skipping the orthogonality loss — without it, the signal and residual encoders learn redundant representations, defeating the purpose of the split architecture.
- **Avoid:** Using PEARL when you have fewer than ~50 labeled examples total or fewer than 3 classes. At that scale, even prototype computation is unreliable.

## Error Handling

| Problem | Symptom | Fix |
|---------|---------|-----|
| Prototype collapse | All prototypes converge to similar vectors | Check that labels are correct and classes are genuinely distinct. Increase label count. |
| Loss divergence | NaN or exploding loss values | Reduce learning rate (try 1e-4). Check for zero-norm embeddings in input. |
| No improvement over raw | Purity@K unchanged after training | Verify labels are accurate. Try increasing `d_signal`. Check that reconstruction loss is decreasing (ensures the network is learning). |
| Degradation at high K | Purity@1 improves but Purity@20 drops | The refiner is overfitting to local structure. Add dropout (0.1-0.2) to encoder layers or reduce training epochs. |
| Class imbalance | Rare classes get worse | Oversample rare classes during training or weight the alignment and contrastive losses by inverse class frequency. |

## Limitations

- **Requires some labels.** PEARL is not unsupervised. With zero labels, use PCA whitening or mean centering instead. PEARL's sweet spot is 100-2500 labels.
- **Assumes class structure exists.** If your task is open-ended semantic similarity without discrete classes (e.g., STS benchmarking), prototype alignment is not applicable.
- **Single-label only.** The prototype formulation assumes each example belongs to one class. Multi-label scenarios require adapting the prototype computation and loss.
- **Frozen base encoder.** PEARL refines post-hoc — it cannot fix fundamental failures of the base encoder (e.g., if the encoder produces identical embeddings for semantically different texts).
- **Domain specificity.** A PEARL refiner trained on one domain's labels will not transfer to a different domain. Retrain the lightweight refiner when the domain changes.
- **Dimensionality reduction.** The signal encoder output has dimension `d_signal < d_in`, which changes the embedding size. Downstream systems expecting the original dimensionality need updating.

## Reference

**Paper:** [PEARL: Prototype-Enhanced Alignment for Label-Efficient Representation Learning](https://arxiv.org/abs/2601.17495v1) — Zhang et al., 2026. Focus on Section 3 (Method) for the six-objective loss formulation and Section 4 (Experiments) for the label-budget ablation showing where gains are largest (100-600 labels).

**Package:** `pip install pearl-H` — [PyPI](https://pypi.org/project/pearl-H/)