---
name: "dart-ing-drift-dynamic-tracing"
description: "Implement DART (Dynamic Attention-Guided Runtime Tracing) for inference-time FFN pruning in LLMs. Dynamically traces knowledge neurons and applies context-aware sparsity masks during autoregressive generation. Use when: 'prune LLM at inference time', 'dynamic neuron masking', 'reduce FFN parameters during generation', 'context-aware model pruning', 'knowledge neuron tracing', 'adaptive sparsity for transformers'."
---

# DART: Dynamic Attention-Guided Runtime Tracing for Adaptive Inference-Time Pruning

This skill enables Claude to implement, configure, and debug DART — a training-free, inference-time pruning method that monitors attention output distributions to detect semantic context shifts, then dynamically updates neuron-level masks in Feed-Forward Network (FFN) layers to retain only context-relevant parameters. DART achieves up to 70% FFN sparsity with minimal accuracy loss and under 10 MB memory overhead, making it practical for deploying large language models under tight resource constraints.

## When to Use

- When the user wants to reduce LLM inference cost by pruning FFN neurons dynamically rather than statically
- When implementing context-aware sparsity that adapts as the model generates tokens autoregressively
- When the user asks to detect "knowledge drift" — shifts in which neurons are important as generation context evolves
- When building an inference pipeline that needs to run LLaMA, GPT-2, or similar models at 50-70% FFN sparsity without calibration data
- When comparing dynamic pruning approaches against static baselines like Wanda or SparseGPT
- When the user needs lightweight neuron importance scoring that avoids gradient computation or dataset-specific calibration

## Key Technique

**Core Insight:** FFN layers in transformers function as key-value memory stores. During autoregressive generation, only a small, context-dependent subset of neurons fires strongly at any given point. Static pruning picks one fixed subset and loses accuracy when the semantic context shifts (e.g., a summarization prompt that spans multiple topics). DART solves this by treating pruning as a dynamic process: it watches the attention outputs for signs that the model's semantic focus has changed, then re-computes which neurons matter.

**Context Drift Detection:** DART computes a semantic centroid of attention outputs from the final transformer layer over a sliding window of tokens. It measures cosine similarity between the current window's centroid and a reference centroid derived from the initial prompt. When similarity drops below a threshold defined by `mean - delta * stddev` (where delta defaults to 0.5), a drift counter increments. Once the counter exceeds a confidence threshold `c0`, DART triggers mask re-computation. This counter mechanism prevents spurious re-pruning from transient noise.

**Neuron Masking with Layer-Aware Sparsity Redistribution:** Neuron importance within each FFN layer is scored by cumulative squared activation magnitude: `s_i = sum(u_{t,i}^2)` over the token window. Top-k neurons survive. Crucially, the sparsity budget is not uniform across layers — DART uses a depth-aware sensitivity metric combining directional change (cosine distance between layer input and FFN output) and magnitude ratio. Early and late layers receive lower sparsity (they are more sensitive), while middle layers absorb more pruning. Sparsity ratios are iteratively redistributed and clipped to `[p_min, p_max]` bounds.

## Step-by-Step Workflow

1. **Register forward hooks on all FFN layers** in the target transformer model. Each hook captures the intermediate activation tensor (the output of the up-projection / gate in the FFN) per token. Use PyTorch `register_forward_hook` on the MLP modules.

2. **Initialize reference centroids from the prompt.** Process the input prompt through the model. Compute the mean of the final-layer attention outputs across all prompt tokens to get the reference centroid `y_bar_{0,T}`. Also compute `K` non-overlapping window centroids from the prompt to derive `mu` (mean cosine similarity) and `sigma` (standard deviation).

3. **Compute initial neuron importance scores.** For each FFN layer `l`, accumulate squared activation magnitudes `s_i = sum(u_{t,i}^2)` across prompt tokens. Rank neurons by score and create binary masks retaining the top-k per layer.

4. **Assign per-layer sparsity budgets using sensitivity analysis.** For each layer, compute the sensitivity score `S(l) = (1 - cos(y, z)) * ||z - y|| / ||y||` where `y` is the layer input and `z` is the FFN output. Apply depth-aware factors: early layers get `alpha_e=0.25` scaling, late layers get `alpha_l=0.35`. Redistribute the global sparsity target across layers inversely proportional to sensitivity, clipping to `[p_min, p_max]`.

5. **Apply masks during generation.** At each forward pass, element-wise multiply the FFN intermediate activations by the binary mask. Masked neurons produce zero output, effectively pruning them from computation.

6. **Monitor drift every `tau` tokens.** Every `tau` generated tokens (typically 10), compute a new window centroid from the last `tau` attention outputs. Calculate cosine similarity against the reference centroid. If `similarity - mu <= -delta * sigma`, increment the drift counter.

7. **Trigger re-pruning when drift counter reaches `c0`.** Reset the counter. Re-score neuron importance using activations from the recent window of tokens. Update masks with new top-k selections. Optionally update the reference centroid to the current window.

8. **Decay the drift counter when no drift is detected.** If the current window does not flag drift, decrement the counter: `c_t = max(0, c_{t-1} - 1)`. This avoids stale counters from earlier transient drifts.

9. **Evaluate pruned model quality.** Run perplexity, MMLU, HellaSwag, ARC, BoolQ, or ROUGE-L benchmarks comparing the DART-pruned model against the dense baseline and static pruning methods. Expect <2% accuracy loss at 50% sparsity, with degradation increasing beyond 70%.

10. **Tune hyperparameters for your deployment.** Key knobs: `delta` (drift sensitivity, lower = more re-pruning), `tau` (window size, smaller = more responsive but higher overhead), `c0` (confidence threshold, higher = fewer false triggers), `p_min/p_max` (sparsity bounds per layer).

## Concrete Examples

**Example 1: Applying DART to LLaMA-3.1-8B for inference**

User: "I want to run LLaMA-3.1-8B with 60% FFN sparsity using dynamic pruning. Set up DART."

Approach:
1. Clone the DART repository and install dependencies (PyTorch 2.2+, transformers 4.57+).
2. Configure `run_experiment.sh` with the model path and pruning parameters.
3. Register activation hooks on all 32 MLP layers.
4. Run initial importance scoring on the prompt, then generate with dynamic masking.

```python
import torch
from transformers import AutoModelForCausalLM, AutoTokenizer

model = AutoModelForCausalLM.from_pretrained("meta-llama/Llama-3.1-8B", torch_dtype=torch.float16, device_map="auto")
tokenizer = AutoTokenizer.from_pretrained("meta-llama/Llama-3.1-8B")

# --- Hook registration ---
activation_cache = {}

def make_hook(layer_idx):
    def hook_fn(module, input, output):
        # Capture gate/up-projection activations for importance scoring
        activation_cache[layer_idx] = output.detach()
    return hook_fn

for idx, layer in enumerate(model.model.layers):
    layer.mlp.register_forward_hook(make_hook(idx))

# --- Initial forward pass to score neurons ---
prompt = "Summarize the key findings of the following research paper:"
inputs = tokenizer(prompt, return_tensors="pt").to(model.device)
with torch.no_grad():
    outputs = model(**inputs)

# --- Compute importance scores per layer ---
masks = {}
target_sparsity = 0.60
for layer_idx, activations in activation_cache.items():
    scores = (activations ** 2).sum(dim=(0, 1))  # sum over batch and sequence
    k = int((1 - target_sparsity) * scores.numel())
    threshold = scores.topk(k).values[-1]
    masks[layer_idx] = (scores >= threshold).float()

# --- Apply masks during generation (simplified) ---
def make_mask_hook(layer_idx):
    def hook_fn(module, input, output):
        return output * masks[layer_idx].unsqueeze(0).unsqueeze(0)
    return hook_fn

# Re-register hooks with masking
for idx, layer in enumerate(model.model.layers):
    layer.mlp.register_forward_hook(make_mask_hook(idx))
```

Output: The model generates text with 60% of FFN neurons masked per layer, using ~6.4 GB less active parameter memory for FFN computations.

---

**Example 2: Detecting knowledge drift during multi-topic summarization**

User: "My model loses quality when summarizing a document that covers multiple distinct topics. How do I detect when the context shifts?"

Approach:
1. Compute reference centroid from the initial prompt tokens' final-layer attention outputs.
2. During generation, compute windowed centroids every 10 tokens.
3. Flag drift when cosine similarity drops below the adaptive threshold.

```python
import torch.nn.functional as F

# After processing prompt, store reference centroid
ref_centroid = outputs.hidden_states[-1].mean(dim=1)  # shape: (1, hidden_dim)

# Compute baseline statistics from K=5 non-overlapping prompt windows
window_centroids = []  # list of centroids from prompt windows
similarities = [F.cosine_similarity(wc, ref_centroid) for wc in window_centroids]
mu = torch.tensor(similarities).mean()
sigma = torch.tensor(similarities).std()

# During generation, every tau=10 tokens:
delta = 0.5
drift_counter = 0
c0 = 3  # confidence threshold

def check_drift(current_window_hidden_states):
    global drift_counter
    window_centroid = current_window_hidden_states.mean(dim=1)
    sim = F.cosine_similarity(window_centroid, ref_centroid)
    if sim - mu <= -delta * sigma:
        drift_counter += 1
    else:
        drift_counter = max(0, drift_counter - 1)
    if drift_counter >= c0:
        drift_counter = 0
        return True  # trigger re-pruning
    return False
```

Output: The drift detector fires ~100 times per 100k tokens in multi-domain text, triggering mask updates that preserve ROUGE-L scores at 3x the level of static pruning.

---

**Example 3: Layer-aware sparsity redistribution**

User: "How do I assign different sparsity rates to different layers instead of uniform pruning?"

Approach:
1. Compute per-layer sensitivity using directional and magnitude metrics.
2. Apply depth-aware scaling factors (early/late layers get less pruning).
3. Iteratively redistribute the global sparsity budget.

```python
def compute_layer_sensitivity(layer_input, ffn_output):
    """Dual-component sensitivity: directional change * magnitude ratio."""
    cos_sim = F.cosine_similarity(layer_input.flatten(), ffn_output.flatten(), dim=0)
    directional = 1.0 - cos_sim.item()
    magnitude = torch.norm(ffn_output - layer_input) / torch.norm(layer_input)
    return directional * magnitude.item()

def redistribute_sparsity(sensitivities, global_sparsity, p_min=0.3, p_max=0.8):
    """Assign per-layer sparsity inversely proportional to sensitivity."""
    n_layers = len(sensitivities)
    # Depth-aware factors: protect early and late layers
    depth_factors = []
    for i in range(n_layers):
        rel_depth = i / (n_layers - 1)
        if rel_depth < 0.25:
            depth_factors.append(0.25)   # early layers: less pruning
        elif rel_depth > 0.75:
            depth_factors.append(0.35)   # late layers: less pruning
        else:
            depth_factors.append(1.0)    # middle layers: full pruning budget

    weighted = [s * d for s, d in zip(sensitivities, depth_factors)]
    total = sum(weighted)
    sparsities = []
    for w in weighted:
        raw = global_sparsity + (global_sparsity * (1 - w / total))
        sparsities.append(max(p_min, min(p_max, raw)))
    return sparsities
```

Output: Early layers (0-7) get ~40% sparsity, middle layers (8-24) get ~70%, late layers (25-31) get ~50%, preserving model quality at the sensitive boundaries.

## Best Practices

- **Do:** Start with `delta=0.5` and `tau=10` as defaults, then tune `delta` downward if the model loses coherence on multi-topic inputs (more frequent re-pruning helps).
- **Do:** Use the confidence counter (`c0 >= 3`) to prevent thrashing — single-window anomalies should not trigger expensive mask re-computation.
- **Do:** Profile per-layer sensitivity on a representative prompt before setting `p_min` and `p_max` — the optimal range varies by model architecture.
- **Do:** Cache neuron importance scores and reuse them until drift is detected. Recomputing every token negates the efficiency gains.
- **Avoid:** Applying uniform sparsity across all layers. Early embedding-facing layers and the final logit-projecting layers are disproportionately sensitive to pruning.
- **Avoid:** Setting sparsity above 75% for tasks requiring factual recall (QA, MMLU). DART's gains over static pruning are largest at 50-70% sparsity; beyond that, accuracy drops sharply for knowledge-intensive tasks.

## Error Handling

- **NaN activations after masking:** If zeroing out neurons causes downstream LayerNorm instability, ensure masks are applied to the intermediate FFN activations (after the gate), not to the final residual stream. The residual connection should bypass the mask.
- **Drift detector never fires:** If `delta * sigma` is too large relative to actual variation, the threshold is unreachable. Lower `delta` or increase window size `tau` to reduce centroid noise.
- **Drift detector fires constantly:** The confidence counter `c0` is too low or `delta` is too aggressive. Increase `c0` to 5+ or raise `delta` to 1.0.
- **Memory not reduced on GPU:** Binary masks alone don't reduce GPU memory — they only zero activations. For actual memory savings, combine with sparse tensor formats or custom CUDA kernels that skip masked neurons.
- **Hook interference with `torch.compile` or FSDP:** Forward hooks may not be compatible with all torch compilation strategies. Test with `torch.compile(backend="eager")` first, then try `inductor`.

## Limitations

- **No wall-clock speedup without sparse kernels.** DART creates structured zeros in activations, but standard dense matrix multiplications don't skip zeros. Real speedup requires sparse CUDA kernels (e.g., cuSPARSE, Triton block-sparse) or hardware with sparsity support (NVIDIA A100 structured sparsity).
- **Prompt-dependent reference centroids.** The drift detector's baseline statistics come from the initial prompt. Very short prompts (< 20 tokens) yield unreliable `mu` and `sigma`, causing false drift triggers.
- **Tested primarily on LLaMA and GPT-2 architectures.** Models with different FFN structures (e.g., mixture-of-experts, GLU variants) may need adaptation of the hook placement and importance scoring.
- **Not suitable for latency-critical single-token generation.** The overhead of centroid computation and drift checking every `tau` tokens adds latency. Best suited for batch inference or throughput-oriented serving.
- **Effectiveness degrades beyond 75% sparsity.** At extreme sparsity, even dynamic re-pruning cannot recover the information lost from removing three-quarters of FFN capacity.

## Reference

**Paper:** [DART: Dynamic Attention-Guided Runtime Tracing of Knowledge Neurons for Adaptive Inference-Time Pruning](https://arxiv.org/abs/2601.22632v1) — Tyagi et al., 2026. Focus on Section 3 (Method) for the drift detection algorithm and sparsity redistribution formulas, and Section 4.2 for ablation studies on `delta`, `tau`, and layer sensitivity.

**Code:** [github.com/seeder-research/DART](https://github.com/seeder-research/DART) — Entry point is `dynamicPrune.py`; core masking logic is in `src/neuronDefuser.py`.