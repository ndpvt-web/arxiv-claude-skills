---
name: "cimrag-cim-aware-domain-adaptive-noise-resilient"
description: "Build noise-resilient RAG retrieval pipelines for edge/resource-constrained deployments. Implements TONEL (Task-Oriented Noise-resilient Embedding Learning) with quantized projections, noise-aware training, and unsupervised domain adaptation. Use when: 'make my RAG work on edge devices', 'add noise robustness to embeddings', 'quantize my retrieval embeddings to INT8', 'domain-adaptive retrieval without labels', 'noise-resilient similarity search', 'CiM-friendly embedding pipeline'."
---

# CiMRAG: Noise-Resilient, Quantized RAG for Edge Deployment

This skill enables Claude to build retrieval-augmented generation pipelines that remain accurate under hardware noise and extreme quantization constraints. Based on the TONEL framework from CiMRAG (ICASSP 2026), it teaches a projection model to compress high-dimensional float embeddings into compact INT8 vectors while injecting simulated device noise during training -- so the retrieval system learns to be robust before it ever hits real hardware. The approach also uses unsupervised pseudo-label clustering for domain adaptation, eliminating the need for manual task labels.

## When to Use

- When the user wants to deploy a RAG system on edge devices, mobile hardware, or memory-constrained environments
- When building retrieval over quantized (INT8) embeddings and needing to preserve ranking quality
- When the user asks to make embedding-based search robust to analog noise, sensor noise, or hardware perturbation
- When adapting a retrieval pipeline to new domains (e.g., medical, legal, travel) without labeled data
- When reducing embedding dimensionality (e.g., 384-d to 64-d) while maintaining retrieval accuracy
- When implementing Maximum Inner Product Search (MIPS) under fixed-precision arithmetic constraints
- When the user mentions Computing-in-Memory (CiM), RRAM, FeFET, or crossbar array deployments

## Key Technique

**The core insight:** Standard RAG embeddings are high-dimensional floating-point vectors that break down when quantized to INT8 or subjected to analog hardware noise. TONEL solves this by training a lightweight projection model that simultaneously (1) compresses embeddings to hardware-compatible dimensions, (2) quantizes them with fake-quantization during training so gradients flow through the rounding operation, and (3) injects calibrated Gaussian noise matching real device characteristics. The model learns embeddings that are inherently noise-tolerant rather than merely hoping full-precision embeddings survive quantization.

**Noise-Aware Task-Oriented Optimization (NATO):** The training loop encodes documents with a frozen pretrained encoder (e.g., `all-MiniLM-L6-v2` producing 384-d vectors), passes them through a trainable linear projection to 64 dimensions, applies simulated INT8 uniform quantization (clamp, scale, round), then adds Gaussian noise `η ~ N(0, σ_v)` calibrated to target hardware (σ_v ranges from 0.004 to 0.015 for real RRAM/FeFET devices). The CiMCE loss trains the projection to produce embeddings where correct documents remain top-ranked even after this quantization + noise pipeline.

**Unsupervised Domain Adaptation via Pseudo-Label Generation (PGM):** Instead of requiring task labels, PGM runs K-means clustering on the pretrained encoder's output embeddings. Each document gets a pseudo-label from its cluster assignment. These pseudo-labels serve as supervision for the CiMCE loss, enabling the projection model to learn task-discriminative structure without any human annotation. This makes the system adaptable to new domains simply by re-clustering.

## Step-by-Step Workflow

1. **Encode the document corpus with a pretrained sentence encoder.** Use a model like `sentence-transformers/all-MiniLM-L6-v2` (384-d output). Freeze this encoder -- it provides the base representations. Store all document embeddings as FP32 numpy arrays.

2. **Generate pseudo-labels via K-means clustering.** Run K-means on the FP32 document embeddings with K set to the number of expected task domains (e.g., K=15 for diverse topics, K=5 for broad categories). Assign each document its cluster ID as a pseudo-label. If ground-truth task labels exist, use those instead for better performance.

3. **Define the projection model.** Implement a single linear layer (`nn.Linear(384, 64)`) that maps from the encoder dimension to the target hardware dimension (64 for a 64x64 crossbar array). This is the only trainable component.

4. **Implement fake quantization in the forward pass.** After the linear projection, apply simulated INT8 quantization: compute a per-tensor scale factor `s = max(|x|) / 127`, then `x_q = clamp(round(x / s), -128, 127) * s`. Use straight-through estimator (STE) so gradients pass through the rounding operation during backpropagation.

5. **Inject calibrated Gaussian noise after quantization.** Add `η ~ N(0, σ_v)` to the quantized embeddings, where σ_v matches target hardware noise levels. Use σ_v=0.01 as a reasonable default; for specific CiM devices, calibrate from hardware measurements (RRAM: ~0.004-0.008, FeFET: ~0.010-0.015).

6. **Train with CiMCE loss.** For each document embedding, compute the noisy quantized projection, then classify it against pseudo-labels using cross-entropy: `L = -mean(sum(y_pseudo * log_softmax(classifier(projected_noisy))))`. The classifier is a simple linear head mapping from 64-d to K classes. Train for 50-100 epochs with Adam (lr=1e-3).

7. **Build the retrieval index with quantized projected embeddings.** After training, project all document embeddings through the trained projection, quantize to INT8 (without noise -- noise was only for training robustness), and store as the retrieval index. Use MIPS (dot product) for similarity scoring.

8. **Process queries through the same pipeline.** Encode user queries with the same frozen encoder, project through the trained projection model, quantize to INT8, then perform MIPS against the document index to retrieve top-K results.

9. **Feed retrieved documents to the edge LLM.** Concatenate the top-K retrieved documents with the user query as context, then pass to a small edge-friendly LLM (Gemma-2B, Llama-3.2-3B, Phi-3-mini) for response generation.

10. **Adapt to new domains by re-clustering.** When the user profile or document corpus shifts domains, re-run K-means on the new embeddings and retrain only the projection model (minutes on CPU). The frozen encoder and LLM remain unchanged.

## Concrete Examples

**Example 1: Building a noise-resilient retrieval pipeline in Python**

User: "I need to deploy RAG on an edge device. My embeddings break when I quantize them to INT8. Help me build a noise-robust retrieval system."

Approach:
1. Encode documents with a frozen sentence transformer
2. Train a TONEL projection with fake quantization and noise injection
3. Build an INT8 retrieval index

Output:
```python
import torch
import torch.nn as nn
import numpy as np
from sentence_transformers import SentenceTransformer
from sklearn.cluster import KMeans

# Step 1: Encode corpus
encoder = SentenceTransformer("all-MiniLM-L6-v2")
documents = ["user profile doc 1...", "user profile doc 2...", ...]
doc_embeddings = encoder.encode(documents, convert_to_numpy=True)  # (N, 384)

# Step 2: Generate pseudo-labels
kmeans = KMeans(n_clusters=10, random_state=42).fit(doc_embeddings)
pseudo_labels = torch.tensor(kmeans.labels_, dtype=torch.long)

# Step 3-5: TONEL projection model with fake quantization + noise
class TONELProjection(nn.Module):
    def __init__(self, in_dim=384, out_dim=64, n_classes=10, sigma_v=0.01):
        super().__init__()
        self.proj = nn.Linear(in_dim, out_dim)
        self.classifier = nn.Linear(out_dim, n_classes)
        self.sigma_v = sigma_v

    def fake_quantize(self, x):
        scale = x.abs().max() / 127.0
        x_q = torch.clamp(torch.round(x / (scale + 1e-8)), -128, 127) * scale
        return x_q + (x_q - x).detach()  # STE: gradient flows through

    def forward(self, x, inject_noise=True):
        projected = self.proj(x)
        quantized = self.fake_quantize(projected)
        if inject_noise and self.training:
            noise = torch.randn_like(quantized) * self.sigma_v
            quantized = quantized + noise
        logits = self.classifier(quantized)
        return logits, quantized

# Step 6: Train
model = TONELProjection(in_dim=384, out_dim=64, n_classes=10, sigma_v=0.01)
optimizer = torch.optim.Adam(model.parameters(), lr=1e-3)
loss_fn = nn.CrossEntropyLoss()

X = torch.tensor(doc_embeddings, dtype=torch.float32)
for epoch in range(80):
    model.train()
    logits, _ = model(X, inject_noise=True)
    loss = loss_fn(logits, pseudo_labels)
    optimizer.zero_grad()
    loss.backward()
    optimizer.step()

# Step 7: Build INT8 index
model.eval()
with torch.no_grad():
    _, proj_embs = model(X, inject_noise=False)
    scale = proj_embs.abs().max() / 127.0
    int8_index = torch.clamp(torch.round(proj_embs / scale), -128, 127).to(torch.int8)

# Step 8: Query
query = "What restaurant did I visit last week?"
q_emb = torch.tensor(encoder.encode([query]), dtype=torch.float32)
with torch.no_grad():
    _, q_proj = model(q_emb, inject_noise=False)
    q_int8 = torch.clamp(torch.round(q_proj / scale), -128, 127).to(torch.int8)

scores = (int8_index.float() @ q_int8.float().T).squeeze()
top_k_indices = scores.topk(5).indices.tolist()
retrieved = [documents[i] for i in top_k_indices]
```

**Example 2: Adapting retrieval to a new domain without labels**

User: "My RAG system was set up for travel queries but now I need it to handle medical questions too. I don't have labeled medical data."

Approach:
1. Add medical documents to the corpus
2. Re-run K-means clustering to discover new domain structure
3. Retrain only the projection model

Output:
```python
# Add new medical documents to existing corpus
medical_docs = ["Patient history for case 42...", "Dosage for medication X...", ...]
all_docs = existing_travel_docs + medical_docs
all_embeddings = encoder.encode(all_docs, convert_to_numpy=True)

# Re-cluster to discover new domain structure (travel + medical)
kmeans_new = KMeans(n_clusters=15, random_state=42).fit(all_embeddings)
new_labels = torch.tensor(kmeans_new.labels_, dtype=torch.long)

# Retrain projection with updated pseudo-labels
model_adapted = TONELProjection(in_dim=384, out_dim=64, n_classes=15, sigma_v=0.01)
X_new = torch.tensor(all_embeddings, dtype=torch.float32)
optimizer = torch.optim.Adam(model_adapted.parameters(), lr=1e-3)

for epoch in range(80):
    model_adapted.train()
    logits, _ = model_adapted(X_new, inject_noise=True)
    loss = loss_fn(logits, new_labels)
    optimizer.zero_grad()
    loss.backward()
    optimizer.step()
# Projection model now handles both travel and medical retrieval
```

**Example 3: Calibrating noise to a specific hardware target**

User: "I'm deploying on an RRAM crossbar array. How do I set the noise parameter?"

Approach:
1. Measure or look up device noise characteristics
2. Set σ_v to match the device's standard deviation
3. Train with hardware-matched noise

Output:
```python
# Known noise profiles from CiMRAG paper (Table I):
DEVICE_NOISE = {
    "rram_low":   0.0038,  # Low-noise RRAM
    "rram_high":  0.0078,  # High-noise RRAM
    "fefet_low":  0.0101,  # Low-noise FeFET
    "fefet_high": 0.0151,  # High-noise FeFET
    "generic":    0.0100,  # Safe default for unknown hardware
}

# Select device profile
sigma_v = DEVICE_NOISE["rram_high"]

# Train with device-matched noise
model = TONELProjection(in_dim=384, out_dim=64, n_classes=10, sigma_v=sigma_v)
# ... training loop as above ...

# For extra robustness, train with slightly higher noise than measured
safety_margin = 1.2
model_robust = TONELProjection(sigma_v=sigma_v * safety_margin)
```

## Best Practices

- **Do:** Always include fake quantization in the training forward pass. Training in FP32 and quantizing at deployment causes a precision cliff -- the model must see quantization during training via STE.
- **Do:** Inject noise during training only, not during inference. Noise injection is a regularization mechanism that teaches robustness; adding noise at inference time degrades results.
- **Do:** Set K (number of clusters) to roughly match the number of distinct task types in your corpus. Too few clusters under-differentiate domains; too many create noisy pseudo-labels.
- **Do:** Use a safety margin of 1.1-1.3x on σ_v relative to measured device noise to account for worst-case environmental conditions (temperature variation, aging).
- **Avoid:** Training the upstream sentence encoder. TONEL is designed to work with a frozen pretrained encoder. Fine-tuning the encoder defeats the purpose of the lightweight projection approach and massively increases training cost.
- **Avoid:** Using cosine similarity with INT8 embeddings. The CiMRAG pipeline is built around Maximum Inner Product Search (MIPS) via dot product, which maps directly to crossbar array multiply-accumulate operations. Normalizing to unit vectors loses information in quantized space.

## Error Handling

- **Quantization collapse (all embeddings map to same INT8 values):** Scale factor is too large. Add a small epsilon to the scale computation and consider reducing the projection output range with a tanh activation before quantization.
- **Pseudo-labels are degenerate (one cluster dominates):** The corpus may lack diversity, or K is too low. Increase K, or use a different clustering algorithm (e.g., spherical K-means for normalized embeddings).
- **Retrieval accuracy drops sharply at high noise (σ_v > 0.015):** The projection dimension may be too low for the noise level. Increase from 64 to 128 dimensions if hardware allows, or reduce noise exposure through hardware shielding.
- **STE gradients vanish:** If the projection outputs are far from integer boundaries, rounding has near-zero effect and training stalls. Initialize the projection with small weights so outputs naturally land near quantization levels.
- **Domain shift causes retrieval of irrelevant documents:** Re-run PGM clustering on the updated corpus and retrain the projection. This is by design -- the lightweight projection retrains in minutes.

## Limitations

- The projection model is linear (single layer). It cannot learn complex nonlinear transformations between the encoder space and the quantized space. For highly heterogeneous corpora, a two-layer MLP with ReLU may be needed, but this diverges from the paper's design.
- INT8 quantization to 64 dimensions means each document is represented by only 64 bytes. This hard ceiling limits expressivity for very large corpora (>100K documents) with fine-grained distinctions.
- Pseudo-label quality depends on the pretrained encoder already producing somewhat separable clusters. If the base encoder is poor for your domain, PGM will generate noisy labels and the projection will underperform.
- The noise model assumes independent Gaussian perturbations per element. Real hardware may exhibit correlated noise patterns (row/column correlations in crossbar arrays) that this approach does not capture.
- This technique optimizes retrieval accuracy, not generation quality. If the edge LLM itself is too small to reason well over retrieved context, improved retrieval alone cannot compensate.

## Reference

**Paper:** [CiMRAG: CiM-Aware Domain-Adaptive and Noise-Resilient Retrieval-Augmented Generation for Edge-Based LLMs](https://arxiv.org/abs/2601.20041v3) (ICASSP 2026). Look for: the NATO training pipeline (Section III-B), the CiMCE loss formulation (Eq. 5-7), the PGM pseudo-label generation procedure (Section III-C), and the device noise calibration table (Table I) mapping real RRAM/FeFET σ_v values.