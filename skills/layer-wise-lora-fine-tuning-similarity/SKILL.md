---
name: "layer-wise-lora-fine-tuning-similarity"
description: "Selectively apply LoRA adapters to only the most important transformer layers using CKA similarity-based layer importance scoring. Cuts trainable parameters by 50% with negligible performance loss. Use when: 'fine-tune with fewer parameters', 'select which layers to apply LoRA to', 'optimize LoRA layer selection', 'reduce LoRA training cost', 'CKA layer importance for fine-tuning', 'layer-wise LoRA configuration'."
---

# Layer-wise LoRA Fine-tuning via CKA Similarity

This skill enables Claude to implement **selective LoRA fine-tuning** where adapters are placed only on the layers that matter most, identified by measuring how much each layer transforms its input using Centered Kernel Alignment (CKA). Instead of attaching LoRA adapters to every transformer layer (the default in most frameworks), this technique runs a single forward pass over your training data, scores each layer's importance as `I = 1 - CKA(input, output)`, ranks them, and applies LoRA only to the top-N layers. The result is a 50% reduction in trainable parameters and 1.17-1.4x training speedup with negligible accuracy loss, validated across encoder-only (RoBERTa, DeBERTa), decoder-only (LLaMA-2, Mistral, Gemma), and multimodal (LLaVA) architectures.

## When to Use

- When the user wants to **fine-tune an LLM with LoRA** and asks how to reduce memory or compute cost without switching to a smaller model.
- When the user asks **which layers to apply LoRA to** or how to configure `target_modules` / `layers_to_transform` in PEFT.
- When fine-tuning is **hitting GPU memory limits** and the user needs to cut parameter count while preserving performance.
- When the user wants to **profile layer importance** before committing to a full LoRA training run.
- When configuring LoRA for a **new downstream task** and there is no prior guidance on which layers matter.
- When the user is working with **PiSSA, rsLoRA, or other LoRA variants** and wants to combine them with selective layer application (the method is orthogonal to the adapter type).

## Key Technique

### CKA-Based Layer Importance Scoring

Standard LoRA attaches low-rank adapters to every transformer layer equally. This paper shows that layers contribute unevenly to task adaptation. The insight: measure how much each layer changes its input representation, and only fine-tune layers with the largest representational shift.

The metric is **Centered Kernel Alignment (CKA)**, which computes normalized similarity between two sets of representations using the Hilbert-Schmidt Independence Criterion (HSIC). For layer `i`, you compute `CKA(R_{i-1}, R_i)` where `R_{i-1}` is the layer's input and `R_i}` is its output. CKA returns a value in [0, 1] where 1 means identical representations. The **importance score** is then `I_i = 1 - CKA(R_{i-1}, R_i)` -- layers that transform their input the most are the most important to fine-tune.

### Why This Works Better Than Heuristics

Common heuristics like "fine-tune only the last N layers" or "alternate layers" are architecture-dependent and task-blind. The CKA approach is both **data-driven** (it uses your actual training distribution) and **cheap** (it requires only forward passes, no gradient computation). On DeBERTa-v3-base, this method outperforms the Fisher Information Matrix (FIM) approach by 7.68 percentage points on average while being orders of magnitude cheaper to compute, since FIM requires gradient calculations.

## Step-by-Step Workflow

### 1. Collect Layer Representations

Run a forward pass of the **frozen pre-trained model** over a representative sample of your training data (a few hundred examples suffice). At each transformer layer, capture the output hidden states. For encoder-only models, extract the `[CLS]` token representation; for decoder-only models, extract the **last token** representation.

### 2. Compute Pairwise CKA Between Adjacent Layers

For each layer `i` (from 1 to L), compute `CKA(R_{i-1}, R_i)` where `R_{i-1}` is the representation entering layer `i` and `R_i` is the representation leaving it. Use linear CKA (kernel = dot product) for efficiency:

```python
import torch

def linear_cka(X, Y):
    """Compute linear CKA between two representation matrices.
    X, Y: (num_samples, hidden_dim) tensors.
    """
    n = X.shape[0]
    H = torch.eye(n, device=X.device) - torch.ones(n, n, device=X.device) / n
    K = X @ X.T
    Q = Y @ Y.T
    HKH = H @ K @ H
    HQH = H @ Q @ H
    hsic_kq = (HKH * HQH).sum() / (n - 1) ** 2
    hsic_kk = (HKH * HKH).sum() / (n - 1) ** 2
    hsic_qq = (HQH * HQH).sum() / (n - 1) ** 2
    return hsic_kq / (torch.sqrt(hsic_kk) * torch.sqrt(hsic_qq) + 1e-8)
```

### 3. Compute Importance Scores

```python
importance = {}
for i in range(1, num_layers):  # skip layer 0 for encoder models
    cka_score = linear_cka(representations[i - 1], representations[i])
    importance[i] = 1.0 - cka_score.item()
```

### 4. Rank and Select Top-N Layers

Sort layers by descending importance. Select the top N layers, where N = total_layers // 2 is the default (50% selection). This can be tuned: fewer layers = more savings, more layers = closer to full LoRA.

```python
sorted_layers = sorted(importance.items(), key=lambda x: x[1], reverse=True)
n_select = num_layers // 2
selected = [layer_idx for layer_idx, _ in sorted_layers[:n_select]]
```

### 5. Configure LoRA with `layers_to_transform`

Use the Hugging Face PEFT library's `layers_to_transform` parameter to restrict LoRA to selected layers:

```python
from peft import LoraConfig, get_peft_model

config = LoraConfig(
    r=128,               # rank (128 for 7B decoder models, 8 for encoder models)
    lora_alpha=128,       # scaling factor (match r for decoder, 2*r for encoder)
    target_modules=["q_proj", "v_proj"],
    layers_to_transform=selected,  # <-- the key parameter
    lora_dropout=0.05,
    bias="none",
    task_type="CAUSAL_LM",
)
model = get_peft_model(base_model, config)
model.print_trainable_parameters()  # should show ~50% of full LoRA params
```

### 6. Fine-tune with Standard Training Loop

Proceed with normal LoRA training (SFT, DPO, etc.). No changes to optimizer, scheduler, or data pipeline are needed.

### 7. Validate and Compare

Compare against full-LoRA baseline on your eval set. Expect less than 0.3 percentage points average drop for 50% parameter reduction.

## Concrete Examples

**Example 1: Fine-tuning LLaMA-2-7B for math reasoning**

User: "I want to fine-tune LLaMA-2-7B on MetaMathQA but I only have a 24GB GPU. Can I reduce the LoRA parameter count?"

Approach:
1. Load LLaMA-2-7B in 4-bit quantization with `bitsandbytes`.
2. Run 500 MetaMathQA samples through the frozen model, collecting hidden states at each of the 32 transformer layers (last-token position).
3. Compute CKA importance for all 32 layers.
4. Select the top 16 layers by importance score.
5. Configure LoRA with `r=128`, `lora_alpha=128`, `layers_to_transform=[2, 5, 7, 9, 12, 14, 16, 18, 20, 22, 24, 25, 27, 28, 30, 31]` (example ranking).
6. Train with standard SFT on MetaMathQA.

Output:
```
Trainable parameters: 83M (vs 167M for full LoRA) -- 50% reduction
GSM8K accuracy: 65.2% (vs 65.8% full LoRA) -- 0.6 pp drop
Training speedup: 1.2x
Peak GPU memory: 18.1 GB (vs 21.4 GB)
```

**Example 2: Profiling DeBERTa-v3-base layer importance for GLUE**

User: "Which layers of DeBERTa should I apply LoRA to for sentiment classification?"

Approach:
1. Load DeBERTa-v3-base (12 layers).
2. Forward-pass 200 samples from the training set, extract `[CLS]` representations at each layer.
3. Compute importance scores for layers 1-11 (skip layer 0 due to centering matrix behavior with embeddings).
4. Print ranked importance table.

Output:
```
Layer Importance Ranking (DeBERTa-v3-base, SST-2):
  Layer 11: 0.342 (highest -- most transformation)
  Layer 10: 0.298
  Layer  9: 0.251
  Layer  7: 0.218
  Layer  8: 0.195
  Layer  6: 0.172
  ---- selection cutoff (top 6) ----
  Layer  5: 0.134
  Layer  4: 0.112
  Layer  3: 0.098
  Layer  2: 0.076
  Layer  1: 0.061

Recommendation: Apply LoRA to layers [6, 7, 8, 9, 10, 11]
Config: LoraConfig(r=8, lora_alpha=16, layers_to_transform=[6, 7, 8, 9, 10, 11])
```

**Example 3: Generating a reusable layer-selection script**

User: "Write me a script that computes CKA layer importance for any causal LM and outputs the PEFT config."

Approach:
1. Write a self-contained Python script that accepts `--model_name`, `--dataset`, `--n_samples`, `--select_ratio` arguments.
2. Loads the model, runs inference, computes CKA, ranks layers.
3. Outputs a JSON file with `layers_to_transform` and recommended LoRA config.

Output: A complete CLI tool (~120 lines) that produces:
```json
{
  "model": "mistralai/Mistral-7B-v0.1",
  "total_layers": 32,
  "selected_layers": [3, 5, 8, 11, 14, 17, 19, 21, 23, 25, 26, 28, 29, 30, 31, 27],
  "selection_ratio": 0.5,
  "importance_scores": {"31": 0.41, "30": 0.38, "...": "..."},
  "peft_config": {
    "r": 128,
    "lora_alpha": 128,
    "layers_to_transform": [3, 5, 8, 11, 14, 17, 19, 21, 23, 25, 26, 28, 29, 30, 31, 27],
    "target_modules": ["q_proj", "v_proj"]
  }
}
```

## Best Practices

**Do:**
- Use a **representative sample** of your actual training data for the CKA profiling pass (500-1000 examples is sufficient; more gives diminishing returns).
- Start with **50% layer selection** (N = total_layers // 2) as the default -- this is the best-validated operating point.
- Keep the LoRA **rank uniform** across all selected layers. The paper tested per-layer rank variation and found no improvement over uniform rank.
- **Re-run CKA profiling** when switching to a substantially different task or dataset, since layer importance is task-dependent.
- Use `torch.no_grad()` during the profiling forward pass to minimize memory overhead.
- Combine this with **quantization** (QLoRA) for maximum memory savings -- the techniques are orthogonal.

**Avoid:**
- Do not include layer 0 (embedding-to-first-transformer) for encoder-only models -- the centering matrix in CKA produces unreliable scores at this boundary.
- Do not use CKA scores as LoRA rank weights (e.g., higher importance = higher rank). Uniform rank across selected layers works better empirically.
- Do not select fewer than 25% of layers without careful validation -- below this threshold performance degrades more steeply.
- Do not skip the profiling step and fall back to heuristics like "last N layers" -- the paper shows CKA selection outperforms all fixed heuristics by 0.2-2.5 percentage points.

## Error Handling

| Problem | Cause | Fix |
|---------|-------|-----|
| CKA returns NaN or 0.0 | Representations are constant or zero-variance (e.g., padding tokens) | Filter to non-padding positions; ensure batch has diverse inputs |
| All layers score nearly equal | Dataset sample too small or too homogeneous | Increase sample size to 500+; use diverse examples |
| `layers_to_transform` ignored | Older PEFT version (<0.6.0) | Upgrade: `pip install peft>=0.6.0` |
| OOM during profiling | Storing all layer representations simultaneously for large models | Process layers in chunks; use float16; reduce batch size |
| Selected layers differ wildly between runs | Random sample variance | Fix the random seed; use a larger sample; average over 3 runs |

## Limitations

- **Profiling requires a forward pass** over real data, so you need access to at least a sample of the training distribution before training starts. This is a non-issue for standard fine-tuning but matters for on-the-fly adaptation scenarios.
- **CKA is a global metric** -- it captures average representational change across the sample. Layers that are critical for rare subsets of the data may be underweighted.
- **Validated at 50% selection**. The paper's strongest results are at the half-layer point. Aggressive pruning (e.g., 75% reduction) shows steeper performance drops and is less predictable.
- **Single-token extraction** (CLS or last token) may miss information for tasks where intermediate token positions are more informative (e.g., span extraction, token classification).
- **Not validated for adapters beyond attention projections** (e.g., MLP layers, layer norms). The paper applies LoRA to q_proj/v_proj; extending to all linear layers may shift the importance rankings.

## Reference

**Paper:** [Layer-wise LoRA fine-tuning: a similarity metric approach](https://arxiv.org/abs/2602.05988v1) (Ogawa et al., 2026). Look for Section 3 (methodology) for the CKA importance formula and Section 4 (experiments) for per-architecture results tables showing which layers are consistently selected.