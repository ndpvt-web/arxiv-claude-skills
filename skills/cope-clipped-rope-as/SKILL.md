---
name: "cope-clipped-rope-as"
description: "Implement CoPE (Clipped RoPE) soft clipping of low-frequency rotary positional embedding components to extend LLM context length without retraining. Use when: 'extend context window with CoPE', 'apply soft clipping to RoPE', 'fix long context degradation', 'implement CoPE positional embedding', 'scale RoPE to longer sequences', 'add cosine-decay frequency clipping'"
---

# CoPE: Soft Clipping of RoPE for Long Context Extension

This skill enables Claude to implement CoPE (Clipped RoPE), a training-free technique that extends the effective context length of RoPE-based LLMs by applying cosine-decay soft clipping to low-frequency positional embedding components. CoPE eliminates out-of-distribution position signal outliers, preserves semantic attention patterns, and avoids spectral leakage artifacts that plague hard-clipping approaches -- yielding gains from 4k to 256k context lengths as a drop-in modification to the RoPE frequency vector.

## When to Use

- When implementing or modifying rotary positional embeddings in a transformer model and the user wants to support longer context than the model was trained on
- When a user reports degraded perplexity, recall, or summarization quality at long context lengths in a RoPE-based model (LLaMA, Qwen, Mistral, etc.)
- When comparing or selecting context extension methods (CoPE vs YaRN vs NTK-aware scaling vs Position Interpolation)
- When building inference pipelines that need to handle sequences beyond the model's pre-training context window without fine-tuning
- When the user asks to "clip RoPE frequencies", "taper low-frequency components", or "fix RoPE extrapolation"
- When integrating long-context support into an existing HuggingFace transformers or vLLM deployment

## Key Technique

**The Problem.** RoPE encodes position information by rotating query/key vectors at dimension-specific frequencies. Lower dimensions rotate fast (high frequency, encoding local position), while higher dimensions rotate slowly (low frequency, encoding global position). When a model encounters sequence positions beyond its training window, these low-frequency components enter out-of-distribution territory, producing unreliable attention scores. Hard-clipping these frequencies (setting them to zero) introduces spectral leakage -- sinc-kernel oscillations with slow O(1/tau) decay that corrupt the attention pattern.

**The CoPE Solution.** Instead of hard-clipping, CoPE applies a cosine-decay taper to the last N entries of the inverse-frequency vector (`inv_freq`). The weight function is `w = 0.5 * (1 + cos(theta))` where theta sweeps from 0 to pi across the clipped dimensions. This smoothly attenuates low-frequency components from full strength to zero, eliminating OOD outliers while preventing Gibbs oscillations. The modification touches only the `inv_freq` initialization -- no architectural changes, no attention mask modifications, fully compatible with FlashAttention.

**Why it works.** CoPE unifies two previously separate goals: (1) OOD mitigation -- the tapered frequencies never produce extreme rotation angles at unseen positions; (2) semantic modeling -- by suppressing slow-rotating dimensions that encode positional rather than semantic information, attention scores more reliably reflect token similarity. On Llama-3-8B extended to 64k context, CoPE improves HELMET scores by 10.8% within the training range and nearly doubles performance under 256k extrapolation (14.37% to 28.48%), with zero degradation on short-context benchmarks (MMLU, GSM8K).

## Step-by-Step Workflow

1. **Identify the RoPE implementation.** Locate the `RotaryEmbedding` class (or equivalent) in the model code. In HuggingFace transformers, this is typically in `modeling_llama.py`, `modeling_qwen2.py`, or `modeling_mistral.py`. Find where `inv_freq` is computed -- usually as `inv_freq = 1.0 / (base ** (torch.arange(0, dim, 2).float() / dim))`.

2. **Determine the critical dimension.** Calculate which frequency dimensions correspond to rotation periods exceeding the pre-training context length. The formula is `d_ct = 2 * ceil((d/2) * log_base(L_pre / (2*pi)))` where `d` is head dimension, `base` is the RoPE base frequency, and `L_pre` is the pre-training context length. Dimensions beyond `d_ct` are OOD-prone.

3. **Choose the clip count.** The default is `clip_low_n = 20` (the last 20 entries of `inv_freq`). For a 128-dimensional head (64 frequency entries), this clips ~31% of frequencies. The paper finds clipping ~75% of OOD frequencies optimal. Adjust based on head dimension and how aggressively you need to extrapolate.

4. **Implement the cosine-decay soft mask.** After computing `inv_freq`, apply:
   ```python
   clip = min(clip_low_n, inv_freq.numel())
   if clip > 0:
       start_idx = inv_freq.numel() - clip
       theta = torch.linspace(0.0, torch.pi, steps=clip, device=inv_freq.device, dtype=torch.float32)
       smooth_mask = 0.5 * (1.0 + torch.cos(theta))
       inv_freq = inv_freq.clone()
       inv_freq[start_idx:] = inv_freq[start_idx:] * smooth_mask.to(inv_freq.dtype)
   ```

5. **Handle dynamic frequency updates.** If the model uses dynamic RoPE scaling (e.g., for sequences exceeding `max_position_embeddings`), re-apply the soft mask after each frequency recomputation. This ensures the taper remains active when `inv_freq` is recalculated for longer sequences.

6. **Register the modified inv_freq.** Replace the original `inv_freq` buffer registration with the clipped version. Ensure the mask is applied before `self.register_buffer("inv_freq", inv_freq, persistent=False)`.

7. **Validate with a needle-in-haystack test.** Run a simple retrieval test at 2x and 4x the training context length. CoPE should maintain recall (e.g., RULER NIAH: 60.5% vanilla RoPE vs 78.5% CoPE at 256k). If recall drops, increase `clip_low_n` slightly.

8. **Verify no short-context regression.** Run a standard benchmark (MMLU or similar) to confirm scores remain within noise of the baseline. CoPE should not degrade short-context performance.

9. **Optionally combine with ABF.** CoPE stacks with Adjusted Base Frequency (increasing RoPE base from e.g. 500k to 10M). Apply ABF first to shift frequencies, then apply CoPE soft clipping to the resulting `inv_freq`.

## Concrete Examples

**Example 1: Adding CoPE to a HuggingFace LLaMA model**

User: "I'm serving Llama-3-8B with transformers and it degrades badly past 8k context. Can you add CoPE to extend it?"

Approach:
1. Open the LlamaRotaryEmbedding class in the model's modeling file
2. Add a `use_cope` config flag and clip parameter
3. Apply soft clipping to `inv_freq` after initialization

Implementation -- modify `LlamaRotaryEmbedding.__init__`:
```python
class LlamaRotaryEmbedding(nn.Module):
    def __init__(self, config, device=None):
        super().__init__()
        self.config = config
        self.rope_type = _get_rope_type(config)
        self.max_seq_len_cached = config.max_position_embeddings
        self.original_max_seq_len = config.max_position_embeddings

        inv_freq, self.attention_scaling = ROPE_INIT_FUNCTIONS[self.rope_type](config, device)

        # --- CoPE: soft-clip low-frequency components ---
        use_cope = getattr(config, "use_cope", False)
        self.clip_low_n = getattr(config, "cope_clip_n", 20) if use_cope else 0

        if self.clip_low_n > 0:
            freq_dim = inv_freq.numel()
            clip = min(self.clip_low_n, freq_dim)
            start_idx = freq_dim - clip
            theta = torch.linspace(0.0, torch.pi, steps=clip,
                                   device=inv_freq.device, dtype=torch.float32)
            smooth_mask = 0.5 * (1.0 + torch.cos(theta))
            inv_freq = inv_freq.clone()
            inv_freq[start_idx:] *= smooth_mask.to(inv_freq.dtype)
        # --- end CoPE ---

        self.register_buffer("inv_freq", inv_freq, persistent=False)
        self.original_inv_freq = self.inv_freq
```

Enable it:
```python
from transformers import AutoModelForCausalLM, AutoConfig

config = AutoConfig.from_pretrained("meta-llama/Llama-3-8B")
config.use_cope = True
config.cope_clip_n = 20  # clip last 20 of 64 frequency entries
model = AutoModelForCausalLM.from_pretrained("meta-llama/Llama-3-8B", config=config)
```

**Example 2: Adding CoPE to a custom RoPE implementation**

User: "I have a custom transformer with RoPE. How do I add CoPE soft clipping?"

Approach:
1. Locate where `inv_freq` or `freqs` is computed
2. Apply the cosine taper as a standalone function

Standalone utility:
```python
import torch

def apply_cope_clipping(inv_freq: torch.Tensor, clip_n: int = 20) -> torch.Tensor:
    """Apply CoPE cosine-decay soft clipping to the last clip_n entries of inv_freq.

    Args:
        inv_freq: RoPE inverse frequency tensor, shape (dim // 2,)
        clip_n: Number of low-frequency (high-index) entries to taper.
    Returns:
        Modified inv_freq with soft-clipped low frequencies.
    """
    freq_dim = inv_freq.numel()
    clip = min(clip_n, freq_dim)
    if clip == 0:
        return inv_freq
    start_idx = freq_dim - clip
    theta = torch.linspace(0.0, torch.pi, steps=clip,
                           device=inv_freq.device, dtype=torch.float32)
    smooth_mask = 0.5 * (1.0 + torch.cos(theta))
    inv_freq = inv_freq.clone()
    inv_freq[start_idx:] *= smooth_mask.to(inv_freq.dtype)
    return inv_freq
```

Usage in any RoPE setup:
```python
dim = 128
base = 500000.0
inv_freq = 1.0 / (base ** (torch.arange(0, dim, 2).float() / dim))
inv_freq = apply_cope_clipping(inv_freq, clip_n=20)
```

**Example 3: Choosing clip_n for a different model**

User: "I'm using Qwen2-7B with head_dim=128 and base=1000000. What clip_n should I use?"

Approach:
1. Compute the frequency vector: 64 entries (128 / 2)
2. Calculate the critical dimension for Qwen2's 32k pre-training context
3. Recommend clipping ~75% of OOD frequencies

```python
import math

head_dim = 128
base = 1_000_000
L_pre = 32768  # Qwen2 pre-training context

# Critical frequency index: where rotation period exceeds L_pre
# theta_j = 1 / base^(2j/d), period = 2*pi / theta_j
# Period > L_pre when theta_j < 2*pi / L_pre
theta_crit = 2 * math.pi / L_pre  # ~0.000192

freq_entries = head_dim // 2  # 64
oob_count = 0
for j in range(freq_entries):
    theta_j = 1.0 / (base ** (2 * j / head_dim))
    if theta_j < theta_crit:
        oob_count += 1

# oob_count tells you how many frequencies are OOD
# Clip ~75% of those: clip_n = round(0.75 * oob_count)
clip_n = round(0.75 * oob_count)
print(f"OOD frequencies: {oob_count}, recommended clip_n: {clip_n}")
```

Output: Typically yields `clip_n` between 15-25 depending on the model's base frequency and pre-training length.

## Best Practices

- **Do** apply the soft mask to `inv_freq` at initialization time, not during every forward pass. The mask is static and should only be recomputed if `inv_freq` itself is recalculated (dynamic RoPE scaling).
- **Do** clone `inv_freq` before in-place modification (`inv_freq = inv_freq.clone()`) to avoid corrupting shared tensors in multi-GPU setups or gradient computation.
- **Do** verify the mask shape matches the frequency vector. For head_dim=128, `inv_freq` has 64 entries; for head_dim=64, it has 32 entries. Adjust `clip_n` proportionally.
- **Avoid** hard-clipping (zeroing out frequencies entirely). Hard clipping introduces sinc-kernel spectral leakage with O(1/tau) decay, causing Gibbs oscillations in attention patterns.
- **Avoid** clipping too aggressively (e.g., clip_n > 75% of total frequency entries). This removes in-distribution positional information and degrades short-context performance.
- **Avoid** applying CoPE on top of methods that already heavily modify the frequency spectrum (e.g., full YaRN with temperature scaling). Test the combination carefully; CoPE is designed as a standalone or ABF-complementary method.

## Error Handling

- **No improvement observed:** Verify the mask is actually applied by inspecting `model.model.layers[0].self_attn.rotary_emb.inv_freq` -- the last `clip_n` values should taper toward zero. If they match unmodified values, the config flag is not being read.
- **Short-context regression:** Reduce `clip_n`. If clipping too many in-distribution frequencies, you lose useful positional signal. Start with `clip_n = 10` and increase.
- **NaN or Inf in attention:** Check that `smooth_mask` dtype matches `inv_freq` dtype. Mixed precision mismatches (e.g., float16 inv_freq with float32 mask) can cause issues on some hardware.
- **Dynamic RoPE not re-clipping:** If the model uses `_dynamic_frequency_update` for sequences exceeding `max_seq_len_cached`, ensure the CoPE mask is reapplied after the new `inv_freq` is computed in that method.
- **Multi-GPU inconsistency:** Ensure `inv_freq` is clipped before it is sharded across devices. Apply CoPE in `__init__` before any model parallelism wrapper.

## Limitations

- CoPE is validated primarily on Llama-3-8B. While the technique is architecturally generic to any RoPE-based model, the optimal `clip_n` may differ for models with different head dimensions, base frequencies, or pre-training lengths.
- The method is a positional embedding modification, not a replacement for continued pre-training on long data. For best results at extreme context lengths (128k+), combine CoPE with continued pre-training on long-context data.
- CoPE does not address KV-cache memory constraints. Extending to 256k context still requires sufficient GPU memory for the full KV-cache (or a separate KV-cache compression technique).
- Performance gains are asymmetric: largest improvements appear under extrapolation (beyond training context). Within the training window, gains are moderate (~4-10%).
- The cosine-decay taper shape is fixed. The paper does not explore alternative smooth window functions (Hann, Blackman, Kaiser) which may offer further improvements.

## Reference

**Paper:** [CoPE: Clipped RoPE as A Scalable Free Lunch for Long Context LLMs](https://arxiv.org/abs/2602.05258v1) (Li et al., 2026). Look for Section 3 (the soft clipping weight function and spectral leakage analysis) and Table 1 (HELMET benchmark results across context lengths).

**Code:** [github.com/hrlics/CoPE](https://github.com/hrlics/CoPE) -- reference implementation for Llama-3-8B with training and evaluation scripts.