---
name: "beyond-speedup-utilizing"
description: "Reuse LLM KV caches as free embeddings for confidence scoring and adaptive fast/slow reasoning. Use when: 'extract embeddings from KV cache', 'score answer confidence without extra inference', 'switch between fast and slow thinking', 'build a difficulty classifier from cached keys/values', 'reduce reasoning tokens adaptively', 'chain-of-embedding scoring'."
---

# KV Cache Reuse for Embeddings, Confidence Scoring, and Adaptive Reasoning

This skill teaches Claude to help users implement the technique from "Beyond Speedup -- Utilizing KV Cache for Sampling and Reasoning" (ICLR 2026). The core insight: KV caches produced during normal autoregressive decoding already encode rich contextual information. Instead of discarding them or using them only for decoding speedup, they can be repurposed as free lightweight representations for two practical applications -- **Chain-of-Embedding (KV-CoE)** for answer confidence scoring, and **Fast/Slow Thinking Switching** for adaptive reasoning depth control. This eliminates the need for separate embedding models or extra forward passes.

## When to Use

- When the user wants to extract embeddings from a transformer model without running a separate embedding model, by reusing the KV cache from generation
- When building a confidence estimator that scores generated answers (e.g., detecting likely-wrong math solutions) without requiring multiple samples or extra prompts
- When implementing adaptive reasoning that routes easy questions to fast (no chain-of-thought) mode and hard questions to slow (full reasoning) mode
- When the user asks to reduce token generation costs on reasoning models (Qwen3, DeepSeek-R1) by classifying problem difficulty from the prompt's KV cache
- When designing an LLM serving pipeline that needs lightweight quality signals from already-computed attention states
- When implementing best-of-N selection or rejection sampling and needing a scoring function that costs zero additional FLOPs

## Key Technique

**KV Cache as Representation.** During standard transformer inference, each layer produces key and value tensors of shape `[Batch, Heads, Tokens, HeadDim]`. These are normally kept only to avoid recomputation in autoregressive decoding. This paper shows they can be pooled into fixed-size vectors and used as embeddings. The per-token KV embedding is computed by flattening K and V across heads, concatenating them, and averaging across selected layers:

```
e_t = (1/L) * sum_l( flatten(K^l_t, V^l_t) )   # shape: [H * d_head] or [2 * H * d_head]
```

Three aggregation axes are configurable: **position** (mean, last-token, or first-token pooling across the sequence), **head** (mean or concatenate across attention heads), and **layer** (mean, sum, or concatenate across transformer layers). The best configuration uses the last 4 layers, concatenation across heads, and mean pooling across token positions.

**Chain-of-Embedding (KV-CoE)** tracks how the KV embedding evolves across successive generated tokens. For each consecutive pair of token embeddings, it computes a magnitude shift `delta_r = ||e_{t+1} - e_t||` and an angular shift `delta_theta = arccos(cos_sim(e_{t+1}, e_t))`. These are aggregated into a confidence score: either a real-valued average `CoE-R = mean(alpha * delta_r + beta * delta_theta)` or a complex-modulus score `CoE-C = |mean(delta_r + i * delta_theta)|`. High trajectory turbulence correlates with incorrect answers, enabling rejection of low-confidence outputs.

**Fast/Slow Thinking Switching** trains a lightweight MLP (two linear layers, 512 hidden units, ReLU) on top of pooled KV representations from the *prompt only*. The classifier predicts a continuous difficulty score d in [0, 100]. At inference time, if d > threshold, the model is steered into "slow" reasoning mode (by prepending `<think>` tokens); otherwise it uses "fast" direct-answer mode. Training labels are derived by comparing fast vs. slow outputs against ground truth on math benchmarks (GSM8K, MATH) using a 4-level scheme: 0 = both correct and short, 25 = both correct but slow is long, 75 = only slow correct, 100 = both wrong. This achieves up to 5.7x token reduction with minimal accuracy loss.

## Step-by-Step Workflow

### Application A: KV-CoE Confidence Scoring

1. **Run standard inference with KV cache retention.** Generate the answer with `use_cache=True` and `return_dict_in_generate=True` so the full KV cache is available after generation. Store the `past_key_values` output.

2. **Select layers for embedding extraction.** Choose the last 4 layers (or a budget of ~256 layer-token units). For each selected layer `l`, extract `K^l` and `V^l` tensors of shape `[Batch, Heads, SeqLen, HeadDim]`.

3. **Compute per-token KV embeddings.** For each generated token position `t`, concatenate K and V across the head dimension (yielding `[2 * Heads * HeadDim]` per layer), then average across selected layers to get `e_t`.

4. **Compute trajectory metrics between consecutive tokens.** For each pair `(e_t, e_{t+1})`: calculate magnitude shift `delta_r = L2_norm(e_{t+1} - e_t)` and angular shift `delta_theta = arccos(cosine_similarity(e_t, e_{t+1}))`.

5. **Aggregate into a confidence score.** Use the complex-modulus formulation: `score = abs(mean(delta_r + i * delta_theta))`. Lower scores indicate higher confidence. Apply a threshold (calibrated on a held-out set) to accept or reject the answer.

6. **Use for selection or filtering.** In best-of-N sampling, generate N answers, compute KV-CoE scores for each, and select the answer with the lowest (most confident) score. No extra forward passes are needed since KV caches were already computed.

### Application B: Fast/Slow Thinking Router

1. **Generate baseline data.** Run your reasoning model on a training set (e.g., GSM8K train split) in both fast mode (no `<think>`) and slow mode (full chain-of-thought). Record outputs and correctness for each.

2. **Assign difficulty labels.** For each problem, assign: `d=0` if both modes correct and slow output < 128 tokens; `d=25` if both correct but slow output >= 128 tokens; `d=75` if only slow mode correct; `d=100` if both wrong.

3. **Extract KV features from prompts.** Run each training prompt through the model, capture KV cache from the last 4 layers for the first 64 tokens. Pool features: sum across token positions, concatenate across heads, average across layers. This yields a fixed-size feature vector per prompt.

4. **Train the MLP classifier.** Build a 2-layer MLP (`Linear(input_dim, 512) -> ReLU -> Linear(512, 1)`). Train with MSE loss against difficulty labels, using lr=1e-5, batch_size=1024, for ~500 iterations. Use 90/10 train/validation split.

5. **Deploy the router at inference time.** For each new prompt: run a single forward pass to populate the KV cache, extract and pool features, predict difficulty `d`. If `d > tau` (threshold tuned on validation), prepend `<think>` to trigger slow reasoning; otherwise proceed with fast direct-answer mode.

6. **Optionally implement dynamic switching.** During generation, periodically re-extract KV features and recompute `d`. If the difficulty estimate changes significantly, insert `</think>` to truncate reasoning early or `<think>` to activate deeper reasoning mid-generation.

## Concrete Examples

**Example 1: Building a KV-CoE confidence scorer in PyTorch**

User: "I want to score the confidence of my Qwen2-7B model's math answers using the KV cache, without running extra inference."

Approach:
1. Identify the model's KV cache structure (`past_key_values` is a tuple of `(K, V)` per layer)
2. Write an extraction function that selects the last 4 layers and computes per-token embeddings
3. Implement the trajectory scoring function (complex-modulus variant)
4. Wire it into the generation loop

Output:
```python
import torch
import cmath

def extract_kv_embeddings(past_key_values, layer_indices=(-4, -3, -2, -1)):
    """Extract per-token embeddings from KV cache.
    past_key_values: tuple of (K, V) per layer, each [B, H, T, D]
    Returns: [B, T, 2*H*D] averaged across selected layers.
    """
    embeddings = []
    for li in layer_indices:
        k, v = past_key_values[li]  # each [B, H, T, D]
        # Concatenate K and V along head*dim: [B, T, 2*H*D]
        kv = torch.cat([
            k.transpose(1, 2).flatten(-2),  # [B, T, H*D]
            v.transpose(1, 2).flatten(-2),
        ], dim=-1)
        embeddings.append(kv)
    return torch.stack(embeddings).mean(dim=0)  # [B, T, 2*H*D]

def kv_coe_score(embeddings):
    """Compute KV-CoE-C confidence score from token embeddings.
    embeddings: [B, T, D] tensor of per-token KV embeddings.
    Returns: [B] confidence scores (lower = more confident).
    """
    scores = []
    for t in range(embeddings.shape[1] - 1):
        e_t = embeddings[:, t]
        e_next = embeddings[:, t + 1]
        diff = e_next - e_t
        delta_r = torch.norm(diff, dim=-1)  # [B]
        cos_sim = torch.nn.functional.cosine_similarity(e_t, e_next, dim=-1)
        delta_theta = torch.acos(cos_sim.clamp(-1 + 1e-7, 1 - 1e-7))  # [B]
        scores.append(torch.stack([delta_r, delta_theta], dim=-1))
    scores = torch.stack(scores, dim=1).mean(dim=1)  # [B, 2]
    # Complex modulus: |mean(delta_r + i*delta_theta)|
    return torch.sqrt(scores[:, 0] ** 2 + scores[:, 1] ** 2)
```

**Example 2: Training a fast/slow thinking router**

User: "I have DeepSeek-R1-Distill-Qwen-14B and want to skip chain-of-thought on easy GSM8K-style problems. Help me build the difficulty classifier."

Approach:
1. Generate fast and slow baselines on the training set
2. Assign 4-level difficulty labels
3. Extract KV features from prompt-only forward passes
4. Train the lightweight MLP
5. Deploy with a threshold

Output:
```python
import torch
import torch.nn as nn

class KVDifficultyClassifier(nn.Module):
    def __init__(self, input_dim, hidden_dim=512, dtype=torch.bfloat16):
        super().__init__()
        self.net = nn.Sequential(
            nn.Linear(input_dim, hidden_dim, dtype=dtype),
            nn.ReLU(),
            nn.Linear(hidden_dim, 1, dtype=dtype),
        )

    def forward(self, kv_features):
        """kv_features: [B, Layers, Heads, Tokens, Dim]"""
        # Sum over token positions, concat heads, average layers
        x = kv_features.sum(dim=3)           # [B, L, H, D]
        x = x.flatten(-2)                     # [B, L, H*D]
        x = x.mean(dim=1)                     # [B, H*D]
        return self.net(x).squeeze(-1)        # [B]

def extract_prompt_kv(model, tokenizer, prompt, layer_indices, max_tokens=64):
    """Extract KV features from prompt-only forward pass."""
    inputs = tokenizer(prompt, return_tensors="pt", truncation=True,
                       max_length=max_tokens).to(model.device)
    with torch.no_grad():
        out = model(**inputs, use_cache=True)
    kv_stack = []
    for li in layer_indices:
        k, v = out.past_key_values[li]
        kv_stack.append(torch.cat([k, v], dim=-1))  # [B, H, T, 2D]
    return torch.stack(kv_stack, dim=1)  # [B, Layers, H, T, 2D]

# Training loop sketch
def train_router(model, tokenizer, dataset, labels, epochs=500, lr=1e-5):
    layer_indices = list(range(-4, 0))  # last 4 layers
    # Pre-extract all features
    features = []
    for prompt in dataset:
        kv = extract_prompt_kv(model, tokenizer, prompt, layer_indices)
        features.append(kv.cpu())
    features = torch.cat(features)
    labels = torch.tensor(labels, dtype=torch.bfloat16)

    input_dim = features.shape[2] * features.shape[4]  # H * 2D
    clf = KVDifficultyClassifier(input_dim).to(model.device)
    opt = torch.optim.Adam(clf.parameters(), lr=lr)

    for step in range(epochs):
        idx = torch.randint(0, len(features), (1024,))
        batch_x = features[idx].to(model.device)
        batch_y = labels[idx].to(model.device)
        pred = clf(batch_x)
        loss = nn.functional.mse_loss(pred, batch_y)
        loss.backward()
        opt.step()
        opt.zero_grad()
    return clf
```

**Example 3: Using KV cache as a zero-cost embedding model for MTEB-style tasks**

User: "I already run Llama-3.1-8B for generation. Can I also get sentence embeddings from it without a separate embedding model?"

Approach:
1. After processing any text through the model, extract KV cache from the last 4 layers
2. Pool across positions using mean pooling (masked to actual tokens), concatenate across heads, average across layers
3. L2-normalize the result for cosine similarity tasks

Output:
```python
def kv_embed(model, tokenizer, texts, layer_indices=(-4,-3,-2,-1)):
    """Extract sentence embeddings from KV cache. Zero extra cost if
    you were already running inference on these texts."""
    inputs = tokenizer(texts, return_tensors="pt", padding=True,
                       truncation=True).to(model.device)
    with torch.no_grad():
        out = model(**inputs, use_cache=True)
    mask = inputs["attention_mask"].unsqueeze(-1)  # [B, T, 1]
    embs = []
    for li in layer_indices:
        k, v = out.past_key_values[li]
        kv = torch.cat([k, v], dim=-1)            # [B, H, T, 2D]
        kv = kv.transpose(1, 2)                    # [B, T, H, 2D]
        kv = kv.flatten(-2)                        # [B, T, H*2D]
        # Masked mean pooling
        kv = (kv * mask).sum(dim=1) / mask.sum(dim=1)  # [B, H*2D]
        embs.append(kv)
    emb = torch.stack(embs).mean(dim=0)            # [B, H*2D]
    return torch.nn.functional.normalize(emb, dim=-1)
```

## Best Practices

- **Do** use the last 4 layers for KV extraction -- the paper's ablation shows these carry the strongest signal. Shallow layers contribute noise.
- **Do** concatenate K and V (use `kv_cat` mode) rather than using keys or values alone. The combined representation consistently outperforms either part.
- **Do** calibrate your CoE threshold or difficulty threshold on a held-out set from the same distribution as your deployment data. The optimal threshold varies across models and tasks.
- **Do** keep the KV budget small for the difficulty classifier -- 64 prompt tokens across 4 layers (256 token-layer units) is sufficient and keeps the feature extraction fast.
- **Avoid** using KV embeddings for open-domain retrieval or tasks requiring fine-grained semantic similarity. KV embeddings underperform dedicated embedding models (e.g., 0.59 vs 0.95 on DBpedia classification). They are best suited for scoring/routing within a constrained pipeline.
- **Avoid** training the difficulty classifier on one benchmark and deploying on a very different domain without retraining. The 4-level labeling scheme is tied to the specific fast/slow accuracy gap on your target task.

## Error Handling

- **KV cache shape mismatch across model versions.** Different HuggingFace model implementations return KV caches in different formats (`tuple of tuples` vs `DynamicCache`). Always inspect `type(past_key_values)` and index accordingly. Use `past_key_values[layer_idx]` to get `(K, V)` tuples and verify shapes before pooling.
- **Out-of-memory on long sequences.** KV caches scale linearly with sequence length. For the difficulty classifier, truncate inputs to 64 tokens. For CoE scoring, compute trajectory metrics incrementally during generation rather than storing the full KV tensor.
- **Numerical instability in arccos.** Cosine similarity values near +/-1 cause NaN in `arccos`. Always clamp: `torch.acos(cos_sim.clamp(-1 + 1e-7, 1 - 1e-7))`.
- **Quantized models.** When using 4-bit or 8-bit quantized models (common for 14B+ parameters), the KV cache is still in bfloat16/float16. This works fine for the technique. However, ensure the MLP classifier matches the KV dtype to avoid casting overhead.

## Limitations

- KV-derived embeddings are significantly weaker than purpose-trained embedding models for general NLP tasks. They work well for relative scoring (which answer is more confident?) but poorly for absolute semantic matching.
- The Fast/Slow Thinking classifier requires training data with both fast and slow outputs for your specific task and model. It does not transfer zero-shot across model families.
- Chain-of-Embedding scoring is a single-sample estimator, not a verifier. It detects answer uncertainty but cannot identify *which* reasoning step went wrong.
- The technique is only applicable to decoder-only transformers that expose their KV cache (most HuggingFace models). It does not apply to API-only models (GPT-4, Claude) where KV caches are inaccessible.
- The 5.7x token reduction figure is model and benchmark specific (Qwen3-8B on MATH500 with the generative routing variant, which trades ~4% accuracy).

## Reference

**Paper:** [Beyond Speedup -- Utilizing KV Cache for Sampling and Reasoning](https://arxiv.org/abs/2601.20326v1) (Xing et al., ICLR 2026). Look for: Section 3 for the KV embedding formulation and pooling strategies, Section 4 for Chain-of-Embedding scoring equations, Section 5 for the Fast/Slow classifier architecture and 4-level labeling scheme, and Appendix E for the KV budget ablation.

**Code:** [github.com/cmd2001/ICLR2026_KV-Embedding](https://github.com/cmd2001/ICLR2026_KV-Embedding) -- contains the MTEB evaluation wrapper (`mteb/custom_model.py`), KV classifier training pipeline (`KVClassifier/`), and Chain-of-Embedding inference scripts (`KV-Chain-of-Embedding/`).