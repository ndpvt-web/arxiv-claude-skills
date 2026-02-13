---
name: "beyond-single-perspective-text"
description: "Detect text anomalies (spam, phishing, harmful content) using multi-view embeddings from diverse language models, combining reconstruction error and cross-view contrastive consistency via the MCA² framework. Use when: 'detect anomalous text', 'multi-view text anomaly detection', 'find spam or phishing with embeddings', 'MCA2 anomaly scoring', 'multi-model text classification', 'unsupervised text outlier detection'."
---

# Multi-View Text Anomaly Detection (MCA²)

This skill enables Claude to implement and apply the MCA² (Multi-view Contrastive Anomaly detection with Adaptive allocation) framework for detecting anomalous text. Instead of relying on a single embedding model, MCA² extracts embeddings from multiple pretrained language models (BERT, Qwen, Llama, Stella, etc.), learns per-view autoencoders to capture normal textual patterns, enforces cross-view contrastive consistency so that normal samples align across views while anomalies diverge, and adaptively weights each view's contribution. The result is a robust, unsupervised anomaly detector that generalizes across harmful content moderation, phishing detection, spam filtering, and other text anomaly tasks.

## When to Use

- When the user asks to build an unsupervised text anomaly detection system that doesn't depend on labeled anomaly examples
- When the user wants to detect spam reviews, phishing emails, toxic comments, or other harmful text using embedding-based methods
- When the user has text data and wants to combine multiple embedding models (e.g., BERT + Qwen + Llama) for more robust outlier detection
- When a single embedding model underperforms and the user wants a multi-view ensemble approach
- When the user needs an anomaly scoring pipeline that produces interpretable per-view reconstruction errors and cross-view consistency scores
- When the user is building a content moderation or security filtering pipeline and needs an adaptive detection backbone

## Key Technique

**Core insight:** A single embedding model captures only one "perspective" of text. Different models encode different linguistic features -- BERT captures bidirectional context, Llama captures autoregressive patterns, Qwen captures multilingual nuances. MCA² treats each model's embeddings as a separate "view" and learns what normal text looks like from every perspective simultaneously. Anomalies are texts that look abnormal from *any* perspective or that show inconsistency *across* perspectives.

**Architecture:** MCA² uses a two-stage training pipeline. In Stage 1, per-view autoencoders (encoder-decoder pairs mapping to a shared 128-dim latent space) are trained alongside a cross-view contrastive loss. The contrastive module pulls latent representations of the *same* normal sample from different views together, while pushing apart representations of different samples. This ensures normal texts have consistent multi-view latent codes. In Stage 2, the autoencoders are frozen and a View Importance Gate is optimized -- an MLP that takes PCA-projected batch statistics and outputs softmax-normalized per-view weights, so the system can automatically emphasize the most informative views for each dataset.

**Anomaly scoring:** The final anomaly score for each text sample is `w_recon * reconstruction_score + w_consistency * consistency_score`. Reconstruction score is the mean squared error between original and reconstructed embeddings across all views. Consistency score measures how well the sample's latent codes agree across views using contrastive distance. Normal texts reconstruct well and are consistent; anomalies fail on one or both signals.

## Step-by-Step Workflow

1. **Prepare your text corpus.** Split data into a training set of known-normal text and a test set containing both normal and potentially anomalous text. Store as JSONL with a `text` field and `label` field (0=normal, 1=anomaly for test set; train should be all 0).

2. **Generate multi-view embeddings.** For each text sample, produce embeddings from at least 3 diverse pretrained models. Use models with different architectures for maximum view diversity:
   ```python
   # Example: generate embeddings with sentence-transformers
   from sentence_transformers import SentenceTransformer
   import numpy as np

   models = {
       "bert": SentenceTransformer("bert-base-nli-mean-tokens"),
       "minilm": SentenceTransformer("all-MiniLM-L6-v2"),
       "stella": SentenceTransformer("dunzhang/stella_en_400M_v5"),
   }
   for name, model in models.items():
       embeddings = model.encode(texts, batch_size=64, show_progress_bar=True)
       np.save(f"embeddings/{dataset}/{name}_{split}.npy", embeddings)
   ```

3. **Normalize embeddings per view.** Apply min-max normalization to each view independently so that reconstruction errors are comparable across views with different dimensionalities.

4. **Build per-view autoencoders.** For each view, create an encoder (input_dim -> 512 -> 256 -> 128) and symmetric decoder (128 -> 256 -> 512 -> input_dim) with ReLU activations and batch normalization. All views share the same latent dimension (128).

5. **Implement the contrastive collaboration module.** For each pair of views, compute contrastive loss on L2-normalized latent codes:
   ```python
   def contrastive_loss(z_i, z_j, temperature=0.5):
       z_i, z_j = F.normalize(z_i, dim=1), F.normalize(z_j, dim=1)
       h = torch.cat([z_i, z_j], dim=0)
       sim = torch.matmul(h, h.T) / temperature
       # Positive pairs: (z_i[k], z_j[k]) for same sample k
       # Negative pairs: all other combinations
       labels = torch.arange(z_i.size(0), device=z_i.device)
       labels = torch.cat([labels + z_i.size(0), labels], dim=0)
       mask = torch.eye(h.size(0), device=h.device).bool()
       sim.masked_fill_(mask, -1e9)
       return F.cross_entropy(sim, labels)
   ```

6. **Train Stage 1 (backbone, ~50 epochs).** Optimize autoencoders and contrastive head jointly with combined loss `L = lambda_recon * L_recon + lambda_contrastive * L_contrastive`. Use Adam optimizer with lr=0.002. Keep the view gate frozen (uniform weights).

7. **Train Stage 2 (gate optimization, ~50 epochs).** Freeze all autoencoder parameters. Train only the View Importance Gate MLP with Adam lr=0.001. The gate takes PCA-reduced batch representations and outputs per-view softmax weights. Evaluate AUC each epoch and save the best gate state.

8. **Compute anomaly scores on test data.** For each test sample, compute:
   - Per-view reconstruction MSE, weighted by gate output
   - Cross-view consistency score from contrastive latent distances
   - Final score: `0.3 * weighted_recon + 0.4 * consistency_score`

9. **Rank and threshold.** Sort samples by anomaly score descending. Use AUC-ROC and AUPRC to evaluate detection quality. For production, set a threshold at a desired false-positive rate using the normal validation set's score distribution.

10. **Iterate on view selection.** If performance is low, add more diverse embedding models or remove redundant views. The adaptive gate will automatically downweight unhelpful views, but starting with diverse architectures (one bidirectional, one autoregressive, one multilingual) gives the best foundation.

## Concrete Examples

**Example 1: Spam review detection**

```
User: I have 50K Amazon product reviews, mostly legitimate. I want to detect
fake/spam reviews without labeled spam examples. Can you build a detector?

Approach:
1. Load reviews from JSONL. Use all reviews as training (assume mostly normal).
   Reserve 10% as test set if you have some labeled spam for evaluation.
2. Generate embeddings with 3 models:
   - BERT (bert-base-uncased) for semantic content
   - all-MiniLM-L6-v2 for efficient sentence similarity
   - Qwen2.5-0.5B for multilingual/diverse representation
3. Normalize each view's embeddings to [0, 1] range.
4. Build MCA² model with latent_dim=128, hidden_dims=[512, 256].
5. Train Stage 1 for 50 epochs (lr=0.002, lambda_recon=1.0, lambda_contrastive=1.0).
6. Train Stage 2 for 50 epochs (gate lr=0.001).
7. Score all reviews. Top-scoring samples are likely spam.

Output:
  Review ID  | Anomaly Score | View Weights (BERT/MiniLM/Qwen)
  R-48291    | 0.872         | 0.41 / 0.23 / 0.36
  R-12044    | 0.841         | 0.38 / 0.29 / 0.33
  R-33187    | 0.803         | 0.45 / 0.20 / 0.35
  ...
  AUC-ROC: 0.94 | AUPRC: 0.87 (on labeled test subset)
```

**Example 2: Phishing email detection**

```
User: Build a phishing email detector using the MCA² multi-view approach.
I have 10K legitimate corporate emails for training.

Approach:
1. Preprocess emails: extract subject + body text, strip HTML tags.
2. Generate embeddings with architecturally diverse models:
   - BERT for content semantics
   - Llama-3.2-1B for autoregressive context patterns
   - Stella-400M for nuanced sentence representations
3. Train MCA² on the 10K legitimate emails only (unsupervised).
4. At inference, score incoming emails. Phishing emails will show:
   - High reconstruction error (unusual language patterns)
   - Low cross-view consistency (different models disagree on representation)

Output:
  Email                      | Score  | Recon | Consistency | Flag
  "Urgent: verify account"   | 0.91   | 0.85  | 0.94        | PHISHING
  "Q3 budget review"         | 0.12   | 0.10  | 0.08        | NORMAL
  "You won a prize click..."  | 0.88   | 0.79  | 0.91        | PHISHING
```

**Example 3: Implementing MCA² from the reference repository**

```
User: I want to run MCA² on the OLID hate speech dataset. Walk me through it.

Approach:
1. Clone the repository:
   git clone https://github.com/yankehan/MCA2.git && cd MCA2

2. Install dependencies:
   conda create -n MCA2 python=3.9
   pip install torch sentence-transformers numpy transformers scikit-learn pandas tqdm pyod

3. Generate embeddings (or use pre-computed ones):
   cd embeddings
   python Bert_embedding.py --dataset olid
   python Qwen2.5_embedding.py --dataset olid
   python Llama-3.2-1B_embedding.py --dataset olid

4. Run MCA² evaluation:
   cd ../multiview_two_stage/eval
   python ourmethod_eval.py --dataset olid --seeds 41,42,43,44,45

5. Results are saved to Excel with AUC-ROC and AUPRC per seed,
   plus mean and variance across seeds.

Output:
  Dataset: OLID | Views: BERT+Qwen+Llama
  Seed 41: AUC=0.923, AUPRC=0.871
  Seed 42: AUC=0.931, AUPRC=0.879
  Mean AUC: 0.927 +/- 0.004
```

## Best Practices

- **Do:** Use at least 3 embedding models with different architectures (e.g., one masked LM like BERT, one autoregressive like Llama, one task-tuned like Stella). Architectural diversity maximizes the information gained from multiple views.
- **Do:** Train only on clean, normal data. The autoencoder learns "what normal looks like" -- any anomalies in the training set will degrade reconstruction-based scoring.
- **Do:** Run the two-stage training in order. The gate cannot learn meaningful view weights until the autoencoders produce stable latent representations.
- **Do:** Evaluate with both AUC-ROC (overall ranking quality) and AUPRC (precision at high recall), since anomaly detection datasets are typically imbalanced.
- **Avoid:** Using embedding models that are near-identical (e.g., two BERT variants fine-tuned on the same task). Redundant views waste capacity and don't improve cross-view consistency signals.
- **Avoid:** Skipping min-max normalization. Views with different embedding dimensions (768 vs 1024 vs 4096) will have incomparable reconstruction errors without normalization.
- **Avoid:** Setting both `w_recon` and `w_consistency` weights to extreme values. The defaults (0.3 and 0.4) work well; tune them on a validation set if available.

## Error Handling

- **Embedding dimension mismatch:** If views have different sample counts (e.g., some texts failed to embed), align samples by index before training. Assert `all views have same number of samples` as a precondition.
- **GPU out of memory:** Reduce batch size or use fewer views. Embedding generation can be done on CPU; only MCA² training needs GPU. The model itself is lightweight (~2M parameters for 3 views).
- **Poor AUC despite training:** Check that the training set is truly normal-only. Even 5% contamination can teach the autoencoder to reconstruct anomalous patterns. Also verify that embeddings were normalized per-view.
- **Gate collapses to one view:** This can happen if one view dominates. Lower the gate learning rate (try 0.0005) or increase the temperature parameter in the softmax to encourage more uniform weighting.
- **Contrastive loss not decreasing:** Ensure latent vectors are L2-normalized before computing contrastive similarity. Check the temperature parameter (default 0.5; lower values like 0.1 can be too sharp).

## Limitations

- MCA² is **unsupervised on the anomaly side** -- it learns normal patterns only. It cannot distinguish between different types of anomalies (e.g., spam vs. phishing) without additional supervision.
- Performance depends heavily on the quality and diversity of the pretrained embedding models. If all models share similar blind spots for a particular anomaly type, MCA² will miss it.
- The framework requires pre-computed embeddings, making it a **two-step pipeline** rather than end-to-end trainable. Changes to the text corpus require re-embedding.
- The adaptive gate optimizes on batch-level statistics, so it works best with reasonably large datasets (1K+ samples). Very small datasets may not provide enough signal for meaningful view weighting.
- MCA² has been validated on English-language benchmarks. Multilingual effectiveness depends on the chosen embedding models supporting the target languages.

## Reference

- **Paper:** Liu et al., "Beyond a Single Perspective: Text Anomaly Detection with Multi-View Language Representations," arXiv:2601.17786 (2026). Focus on Section 3 (MCA² framework architecture), Section 4.2 (ablation studies showing contribution of each module), and Table 2 (benchmark results across 10 datasets).
- **Code:** https://github.com/yankehan/MCA2