---
name: "meki-memory-based-expert-knowledge"
description: >
  Implement MeKi (Memory-based Expert Knowledge Injection) — a storage-based LLM scaling
  technique that adds per-token expert knowledge via static lookup tables parallel to FFN layers,
  achieving higher accuracy with zero inference latency overhead. Ideal for edge/mobile deployment.
  Trigger phrases: "scale LLM with memory experts", "add MeKi expert injection", "deploy LLM on
  edge device with expert knowledge", "token-level memory lookup for transformers",
  "re-parameterize expert branch for inference", "ROM-based knowledge offloading".
---

# MeKi: Memory-based Expert Knowledge Injection

This skill enables Claude to implement the MeKi architecture — a system that scales LLM capacity
through storage rather than compute. MeKi adds a lightweight expert branch parallel to each
Transformer FFN layer. During training, the branch learns per-token knowledge via static embedding
tables and dynamic SwiGLU projections. At inference time, all dynamic computations are folded into
a single pre-computed lookup table per layer, indexed directly by token ID and stored in ROM.
This yields measurable accuracy gains (2-4 points on standard benchmarks) with virtually zero
additional latency, making it the primary technique for boosting on-device LLM performance
without increasing RAM or NPU load.

## When to Use

- When the user needs to add a knowledge-injection branch to an existing Transformer model to
  improve accuracy without increasing inference cost.
- When deploying an LLM on mobile/edge hardware (phones, embedded NPUs) and storage is plentiful
  but RAM and compute are constrained.
- When implementing a MoE-like capacity expansion that avoids dynamic routing overhead entirely.
- When the user asks to re-parameterize training-time projections into static lookup tables for
  faster inference.
- When building a pre-training pipeline that includes per-layer, per-token memory expert modules.
- When optimizing an existing dense LLM to match a model 2-3x its size in accuracy while
  maintaining identical decoding speed.

## Key Technique

**Storage-based scaling via token-level memory experts.** Traditional LLM scaling increases
parameters (requiring more RAM and FLOPs) or test-time compute (requiring more NPU cycles). MeKi
introduces a third axis: storage. Each Transformer layer gets a memory embedding table
`M^l` of shape `[vocab_size, d_mem]` that maps every token ID to a learned expert vector. During
training, this static lookup is combined with a dynamic SwiGLU projection `G^l` operating on
shared global embeddings, weighted by learnable scalars `alpha` and `beta`. The combined expert
vector is gated by the current hidden state and added to the residual stream in parallel with the
standard FFN output.

**Re-parameterization eliminates inference overhead.** The key insight is that the dynamic
projection `G^l(E_global[x_t])` depends only on the token ID, not on the sequence context. After
training, MeKi pre-computes `M_tilde^l = alpha^l * RMSNorm(M^l + beta^l * G^l(E_global))` for
every token and every layer, producing a single dense lookup table. At inference time, the entire
expert branch reduces to: (1) index into `M_tilde^l` by token ID, (2) compute a small gate from
the hidden state, (3) project back to model dimension — just two `d_mem x d_model` matmuls and
an addition. The lookup table lives in ROM and can be asynchronously prefetched using the already-
known token ID, so it adds zero wall-clock latency on modern mobile SoCs with fast storage (e.g.,
UFS-4.0 at 4.2 GB/s).

**Practical impact.** MeKi-1.7B matches or exceeds the accuracy of a 4B dense baseline while
decoding 2.26x faster on Snapdragon 8 Elite. The ROM overhead is modest: ~14 KB per token in
float16 for a 28-layer model with `d_mem=256`.

## Step-by-Step Workflow

1. **Define the MeKi branch module.** For each Transformer layer `l`, create: a static embedding
   table `M^l` of shape `[vocab_size, d_mem]`, a SwiGLU projection `G^l` mapping `d_model` to
   `d_mem`, learnable scalars `alpha^l` and `beta^l`, a gate projection `W_gate^l` of shape
   `[d_mem, d_model]`, and an output projection `W_out^l` of shape `[d_model, d_mem]`.

2. **Wire the branch parallel to FFN.** In the Transformer forward pass, after attention output
   `A` and before the residual addition, compute both the standard FFN output `F(H)` and the MeKi
   expert output `MeKi(H)`. The final residual is `H' = F(H) + MeKi(H) + A`.

3. **Implement the training-time forward pass.** For each token `x_t` at layer `l`:
   - Look up static vector: `m_static = M^l[x_t]`
   - Compute dynamic vector: `m_dyn = G^l(E_global[x_t])` (SwiGLU on shared word embeddings)
   - Combine: `e_t = alpha^l * RMSNorm(m_static + beta^l * m_dyn)`
   - Gate: `g_t = sigmoid(W_gate^l @ h_t^l)`
   - Modulate: `v_t = e_t + g_t`
   - Project: `y_t = RMSNorm(W_out^l @ v_t)`

4. **Train end-to-end with standard LM objective.** Use AdamW (beta1=0.9, beta2=0.95), cosine LR
   schedule with warmup, BFloat16 mixed precision. The MeKi parameters train jointly with the base
   model. No special loss terms are needed — standard cross-entropy suffices.

5. **Re-parameterize for inference.** After training completes, pre-compute the merged table for
   every layer: `M_tilde^l[v] = alpha^l * RMSNorm(M^l[v] + beta^l * G^l(E_global[v]))` for all
   token IDs `v` in the vocabulary. Discard `G^l`, `alpha^l`, `beta^l`, and `E_global` — they are
   no longer needed.

6. **Serialize lookup tables to ROM format.** Save each `M_tilde^l` as a contiguous float16
   tensor of shape `[vocab_size, d_mem]`. Total storage per layer is `vocab_size * d_mem * 2`
   bytes. For a 32K vocabulary and `d_mem=256`, that is ~16 MB per layer.

7. **Implement the inference-time forward pass.** For each token `x_t` at layer `l`:
   - Prefetch `M_tilde^l[x_t]` from ROM (async, overlapped with attention compute)
   - Gate: `g_t = sigmoid(W_gate^l @ h_t^l)`
   - Modulate: `v_t = M_tilde^l[x_t] + g_t`
   - Project: `y_t = RMSNorm(W_out^l @ v_t)`
   - Add to residual stream alongside FFN output

8. **Deploy on edge hardware.** Place lookup tables on device storage (ROM/UFS). Keep only
   `W_gate` and `W_out` (small `d_mem x d_model` matrices) in RAM alongside the base model.
   Use the storage controller's prefetch API to load the next token's expert vectors while the
   current attention layer executes.

9. **Validate correctness.** Compare perplexity and zero-shot benchmark scores (ARC, BoolQ,
   HellaSwag, LAMBADA, PIQA, WinoGrande) between the training-time model and the re-parameterized
   inference model. They must match exactly (up to floating-point precision).

10. **Profile and tune `d_mem`.** Sweep `d_mem` in {128, 256, 512}. Larger values increase storage
    and accuracy; smaller values reduce ROM footprint. Choose based on target device storage budget
    and acceptable accuracy gain.

## Concrete Examples

**Example 1: Adding MeKi to a HuggingFace model**

User: "I have a 1.7B parameter GPT-style model. Add a MeKi expert branch to improve accuracy
without increasing inference latency."

Approach:
1. Subclass the model's TransformerBlock to add a `MeKiExpert` module.
2. Initialize `M^l` as `nn.Embedding(vocab_size, d_mem)` and `G^l` as a SwiGLU MLP.
3. In `forward()`, compute expert output parallel to FFN and sum into the residual.

Output (PyTorch module skeleton):
```python
class MeKiExpert(nn.Module):
    def __init__(self, vocab_size, d_model, d_mem=256):
        super().__init__()
        self.static_mem = nn.Embedding(vocab_size, d_mem)
        # SwiGLU: two linear projections with gated activation
        self.gate_proj = nn.Linear(d_model, d_mem, bias=False)
        self.up_proj = nn.Linear(d_model, d_mem, bias=False)
        self.alpha = nn.Parameter(torch.ones(1))
        self.beta = nn.Parameter(torch.ones(1))
        self.hidden_gate = nn.Linear(d_model, d_mem, bias=False)
        self.out_proj = nn.Linear(d_mem, d_model, bias=False)
        self.norm = RMSNorm(d_mem)
        self.out_norm = RMSNorm(d_model)

    def forward(self, token_ids, hidden_states, global_embeddings):
        # Static lookup
        m_static = self.static_mem(token_ids)               # [B, T, d_mem]
        # Dynamic SwiGLU projection on global word embeddings
        emb = global_embeddings(token_ids)                   # [B, T, d_model]
        m_dyn = F.silu(self.gate_proj(emb)) * self.up_proj(emb)
        # Combine and normalize
        e = self.alpha * self.norm(m_static + self.beta * m_dyn)
        # Gate by current hidden state
        g = torch.sigmoid(self.hidden_gate(hidden_states))
        v = e + g
        return self.out_norm(self.out_proj(v))
```

**Example 2: Re-parameterizing for edge deployment**

User: "Training is done. Convert the MeKi branches to static lookup tables for mobile inference."

Approach:
1. Iterate over every layer and every token in the vocabulary.
2. Pre-compute the merged expert vector and write to a flat tensor file.
3. Strip training-only parameters from the checkpoint.

Output (conversion script):
```python
def reparameterize_meki(model, global_embeddings):
    """Convert trained MeKi branches to static lookup tables."""
    tables = {}
    vocab_size = global_embeddings.num_embeddings

    for layer_idx, layer in enumerate(model.layers):
        expert = layer.meki_expert
        all_embs = global_embeddings.weight  # [V, d_model]

        # Compute dynamic projection for all tokens at once
        m_dyn = F.silu(expert.gate_proj(all_embs)) * expert.up_proj(all_embs)
        m_static = expert.static_mem.weight  # [V, d_mem]

        # Merge into single lookup table
        merged = expert.alpha * rms_norm(m_static + expert.beta * m_dyn)
        tables[f"layer_{layer_idx}"] = merged.half().contiguous()

        # Remove training-only params from state dict
        del expert.gate_proj, expert.up_proj
        del expert.alpha, expert.beta, expert.static_mem

    return tables  # Save each as a flat binary file for ROM loading

# Serialize tables
for name, table in tables.items():
    table.numpy().tofile(f"rom/{name}_meki_table.bin")
```

**Example 3: Profiling ROM bandwidth requirements**

User: "Will MeKi fit on my device? I have a 32K vocab model with 28 layers and want d_mem=256."

Approach:
1. Calculate per-layer table size: `32768 * 256 * 2 bytes = 16 MB`.
2. Calculate total ROM: `28 * 16 MB = 448 MB`.
3. Calculate per-token fetch: `28 * 256 * 2 = 14 KB` (one vector per layer).
4. At 30 tokens/sec, bandwidth needed: `30 * 14 KB = 420 KB/s` (trivial for UFS-4.0 at 4.2 GB/s).

Output:
```
MeKi ROM Budget for 28-layer, 32K-vocab, d_mem=256:
  Per-layer table:     16.0 MB
  Total ROM:          448.0 MB
  Per-token fetch:     14.0 KB
  Bandwidth @ 30 tok/s: 0.4 MB/s  (0.01% of UFS-4.0 capacity)
  In-RAM overhead:     W_gate + W_out per layer = 2 * 256 * d_model * 2 bytes
                       For d_model=2048: 2.0 MB/layer, 56 MB total

Verdict: Easily fits. Storage is the only constraint, not bandwidth or RAM.
```

## Best Practices

- **Do:** Initialize `alpha` and `beta` to 1.0 so the expert branch contributes from the start of
  training. Zero-initialization causes the branch to be ignored and never learn.
- **Do:** Use additive gated fusion (sigmoid gate + addition) for injecting expert vectors. The
  paper tested concatenation, replacement, and multiplicative fusion — additive gated was best.
- **Do:** Share global word embeddings (`E_global`) across all layers for the dynamic projection.
  Per-layer embeddings waste parameters without improving accuracy.
- **Do:** Prefetch expert vectors asynchronously using the token ID, which is known before the
  layer computation begins. This hides ROM latency entirely.
- **Avoid:** Skipping the re-parameterization step. Running the full training-time computation at
  inference defeats the purpose — the SwiGLU projection adds significant FLOPs that should be
  folded into the lookup table.
- **Avoid:** Using very large `d_mem` values (>512) without profiling. ROM size scales linearly
  with `d_mem * vocab_size * num_layers`, and diminishing returns set in quickly.

## Error Handling

- **Numerical mismatch after re-parameterization:** If inference results diverge from training-time
  outputs, check that RMSNorm epsilon values match, and that the merged table was computed in
  float32 before casting to float16. Premature quantization during the merge causes drift.
- **ROM prefetch misses:** If latency increases on-device, verify that token IDs are available to
  the prefetch scheduler before the layer begins execution. In autoregressive decoding this is
  guaranteed, but in prefill (parallel prompt processing) you must batch-load all token vectors
  for the layer at once.
- **Training instability:** If loss spikes when adding MeKi to a pre-trained model, reduce the
  initial learning rate for `alpha` and `beta` by 10x relative to other parameters. The gating
  mechanism can amplify gradients early in fine-tuning.
- **Vocabulary mismatch:** The lookup table is vocabulary-specific. If the tokenizer changes, the
  entire table must be retrained. There is no graceful fallback for out-of-vocabulary tokens.

## Limitations

- MeKi provides the largest gains on knowledge-intensive tasks (ARC, LAMBADA, SciQ) and smaller
  gains on reasoning-heavy tasks. It injects factual knowledge per token, not reasoning patterns.
- The approach requires full pre-training or extensive continued pre-training to learn meaningful
  token-level expert vectors. It cannot be bolted onto a model with a short fine-tuning run.
- Storage requirements scale with `vocab_size * d_mem * num_layers`. Models with very large
  vocabularies (100K+) may need `d_mem` reduction or vocabulary partitioning.
- Re-parameterization is a one-way conversion. You cannot resume training from inference-mode
  weights — keep the training checkpoint separately.
- The technique is complementary to, not a replacement for, MoE or other capacity-scaling methods.
  MeKi targets a different bottleneck (storage vs. compute) and can be combined with MoE.

## Reference

- **Paper:** [MeKi: Memory-based Expert Knowledge Injection for Efficient LLM Scaling](https://arxiv.org/abs/2602.03359v1)
  (Ding et al., 2026). Focus on Section 3 (method), Section 3.3 (re-parameterization), and
  Table 1 (benchmark results) for implementation details.