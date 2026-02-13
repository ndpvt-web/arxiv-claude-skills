---
name: "cofrgenet-continued-fraction-architectures"
description: "Implement Continued Fraction Generative Networks (CoFrGeNets) -- parameter-efficient replacements for Multi-head Attention and Feed-Forward Networks in Transformer blocks using continued-fraction-based function classes. Trigger phrases: 'build a continued fraction layer', 'replace attention with CoFrGeNet', 'implement CFA module', 'parameter-efficient transformer alternative', 'continued fraction attention', 'CoFrGeNet architecture'"
---

# CoFrGeNet: Continued Fraction Architectures for Language Generation

This skill enables Claude to implement Continued Fraction Generative Networks (CoFrGeNets), a family of neural network components that replace Multi-head Attention (MHA) and Feed-Forward Networks (FFN) in Transformer blocks. CoFrGeNets use a continued-fraction function class where the reciprocal operation (1/x) serves as the nonlinearity, achieving competitive or superior language modeling performance with 34-47% fewer parameters than standard Transformers. Claude can build CFA (Continued Fraction Attention) modules, CFFN (Continued Fraction Feed-Forward) modules, implement the continuant-based forward/backward passes, and integrate these into existing Transformer training pipelines.

## When to Use

- When the user asks to build a parameter-efficient alternative to standard Transformer attention or FFN layers
- When the user wants to implement continued-fraction-based neural network layers in PyTorch
- When the user needs to reduce model size (e.g., from 1.5B to ~800M-1B parameters) while maintaining language modeling quality
- When the user asks about novel nonlinearities beyond ReLU/GELU -- specifically the reciprocal (1/x) nonlinearity
- When the user wants to replace MHA or FFN in an existing GPT-2 or Llama-style architecture with lighter components
- When the user is experimenting with non-attention sequence modeling architectures
- When the user asks to implement custom autograd functions with analytically-derived gradients for numerical stability

## Key Technique

**Continued fractions as a function class.** A continued fraction has the form `f(a0, a) = a0 + 1/(a1 + 1/(a2 + ... + 1/ad))` where each partial denominator `ak = wk * x` is an affine function with learnable weight `wk`. The depth `d` controls expressiveness. An ensemble of `L` such "ladders" (independent continued fractions) allows the model to capture diverse input-output relationships. The key nonlinearity is the reciprocal operation `1/x` applied at each depth level, which creates rational-function approximations far more expressive per parameter than standard polynomial activations.

**Efficient computation via continuants.** Rather than computing `d` nested divisions (numerically unstable), CoFrGeNet uses continuant polynomials: `K0 = 1, K1 = a_d`, and `Kk = a_{d-k+1} * K_{k-1} + K_{k-2}`. The continued fraction value is then `K_{d-1} / K_d` -- a single division. Custom gradients are derived analytically: `df/da_k = (-1)^k * (K_{d-k} / K_d)^2`, requiring only one reciprocal computation of `K_d` reused across all partials. This is more numerically stable and efficient than PyTorch autograd's chain of reciprocals.

**Two attention replacements and one FFN replacement.** *CFA-U (CAttnU)* transposes the input across embedding and sequence dimensions, then applies upper-triangular linear maps to maintain causality, with element-wise multiplication of two ladder ensembles producing interaction terms. *CFA-M (CAttnM)* generates attention-like weights from `L` ladders mapped to sequence-length space, then causally weights value matrices. *CFFN* replaces the standard expand-and-contract FFN with an ensemble of `p`-variate ladders at expansion factor `alpha=1` (no expansion), gated and projected back. All three are drop-in replacements requiring no changes to training or inference pipelines.

## Step-by-Step Workflow

1. **Define the continuant forward pass as a custom `torch.autograd.Function`.** Implement `K0=1, K1=a_d` and the recurrence `Kk = a_{d-k+1} * K_{k-1} + K_{k-2}` for `k=2..d`. Store all `K` values for the backward pass. Return `K_{d-1} / K_d` (clamping `K_d` away from zero by epsilon=0.01 to avoid poles).

2. **Implement the custom backward pass using the analytical gradient.** Compute `grad_a_k = (-1)^k * (K_{d-k} / K_d)^2 * grad_output`. This avoids the numerical instability of autograd differentiating through nested reciprocals.

3. **Build a single "ladder" module.** A ladder takes input `x` of dimension `p`, applies `d` learnable affine transforms `a_k = W_k @ x` (each `W_k` is a `p x p` or diagonal weight matrix), and passes the resulting sequence through the continuant function. Output is a `p`-dimensional vector.

4. **Build the ladder ensemble.** Stack `L` independent ladders. Each ladder has its own weights. The ensemble output is the concatenation or element-wise combination of all `L` ladder outputs, depending on the component (CFA vs CFFN).

5. **Implement CAttnU (Continued Fraction Attention - Univariate).** Transpose the input tensor from `(batch, seq_len, embed_dim)` to `(batch, embed_dim, seq_len)`. Apply two independent ensembles of univariate ladders with upper-triangular weight matrices (to preserve causal masking). Multiply the two ensemble outputs element-wise to produce cross-term interactions. Transpose back. Parameter count: `L * (2d + L + 1)`.

6. **Implement CAttnM (Continued Fraction Attention - Multivariate).** Keep the input in `(batch, seq_len, embed_dim)` form. Use `L` multivariate ladders to generate attention weight matrices of shape `(seq_len, seq_len)`. Apply causal masking (lower-triangular) to these weights. Multiply by a value projection of the input. Parameter count: `L * (p + seq_len) + p^2`.

7. **Implement CFFN (Continued Fraction Feed-Forward Network).** Apply a gating linear layer to the input (no expansion, `alpha=1`). Pass the gated representation through an ensemble of `L` multivariate ladders of depth `d`. Project the output back to embedding dimension with a final linear layer. Parameter count: `L * p * (d + 1) + 2 * p^2`.

8. **Apply the dyadic training schedule.** During pre-training, parameters at depth `i` in each ladder are updated for only `T / 2^i` iterations (where `T` is total training steps). The linear (depth-0) component trains for the full duration. This coarse-to-fine schedule stabilizes training of deeper ladder levels.

9. **Integrate into an existing Transformer.** Replace `nn.MultiheadAttention` with CAttnU or CAttnM. Replace the FFN sublayer with CFFN. Keep layer normalization, residual connections, embeddings, and positional encodings unchanged. The rest of the Transformer block remains identical.

10. **Validate by comparing perplexity and downstream task performance.** Train on a standard corpus (e.g., OpenWebText). Compare perplexity on PTB, WikiText-2, and Lambada against the baseline Transformer. Expect competitive perplexity at 50-66% of the original parameter count.

## Concrete Examples

**Example 1: Implementing the continuant custom autograd function**

User: "Implement the core continued fraction forward and backward pass in PyTorch"

Approach:
1. Create a `torch.autograd.Function` subclass
2. Forward: compute continuant sequence, return ratio
3. Backward: use analytical gradient formula

Output:
```python
import torch
from torch.autograd import Function

class ContinuantFraction(Function):
    """Continued fraction via continuant polynomials with custom gradient."""

    EPSILON = 0.01  # pole avoidance clamp

    @staticmethod
    def forward(ctx, *partials):
        """
        Args:
            partials: d tensors, each of shape (batch, dim), representing
                      a_1, a_2, ..., a_d (the partial denominators).
        Returns:
            Continued fraction value K_{d-1} / K_d of shape (batch, dim).
        """
        d = len(partials)
        # Continuant recurrence: K_0=1, K_1=a_d, K_k = a_{d-k+1}*K_{k-1} + K_{k-2}
        K = [None] * (d + 1)
        K[0] = torch.ones_like(partials[0])
        K[1] = partials[d - 1]  # a_d

        for k in range(2, d + 1):
            K[k] = partials[d - k] * K[k - 1] + K[k - 2]

        # Clamp K_d away from zero to avoid division-by-zero poles
        K_d_safe = K[d].clamp(min=ContinuantFraction.EPSILON)
        result = K[d - 1] / K_d_safe

        ctx.save_for_backward(K_d_safe, *K)
        ctx.d = d
        return result

    @staticmethod
    def backward(ctx, grad_output):
        saved = ctx.saved_tensors
        K_d_safe = saved[0]
        K = list(saved[1:])
        d = ctx.d

        grads = []
        inv_Kd_sq = 1.0 / (K_d_safe * K_d_safe)
        for k in range(d):
            # df/da_k = (-1)^k * (K_{d-k} / K_d)^2
            sign = 1.0 if k % 2 == 0 else -1.0
            K_dk = K[d - k]
            grad_ak = sign * (K_dk * K_dk) * inv_Kd_sq * grad_output
            grads.append(grad_ak)
        return tuple(grads)


def continued_fraction(partials):
    """Convenience wrapper: partials is a list of d tensors."""
    return ContinuantFraction.apply(*partials)
```

**Example 2: Building a CFFN layer to replace a standard Transformer FFN**

User: "Replace the FFN in my GPT-2 block with a continued fraction feed-forward network"

Approach:
1. Define a single ladder module using the continuant function
2. Build an ensemble of L ladders
3. Wrap with input gating and output projection

Output:
```python
import torch
import torch.nn as nn

class Ladder(nn.Module):
    """Single continued fraction ladder of depth d over p-dim input."""

    def __init__(self, dim, depth):
        super().__init__()
        self.depth = depth
        # Learnable affine weights for each partial denominator
        self.weights = nn.ParameterList([
            nn.Parameter(torch.randn(dim) * 0.02) for _ in range(depth)
        ])

    def forward(self, x):
        # x: (batch, seq_len, dim)
        partials = [w * x for w in self.weights]  # each (batch, seq_len, dim)
        return continued_fraction(partials)


class CFFN(nn.Module):
    """Continued Fraction Feed-Forward Network.
    Drop-in replacement for standard Transformer FFN.
    """

    def __init__(self, embed_dim, num_ladders=8, depth=5):
        super().__init__()
        self.gate_proj = nn.Linear(embed_dim, embed_dim)
        self.ladders = nn.ModuleList([
            Ladder(embed_dim, depth) for _ in range(num_ladders)
        ])
        self.out_proj = nn.Linear(embed_dim, embed_dim)

    def forward(self, x):
        # x: (batch, seq_len, embed_dim)
        gated = torch.sigmoid(self.gate_proj(x)) * x
        # Ensemble: average over L ladder outputs
        ladder_out = torch.stack(
            [ladder(gated) for ladder in self.ladders], dim=0
        ).mean(dim=0)
        return self.out_proj(ladder_out)
```

**Example 3: Integrating CFA into an existing Transformer block**

User: "Show me how to swap multi-head attention for continued fraction attention in a Transformer block"

Approach:
1. Implement CAttnU with upper-triangular causal masking
2. Replace the attention sublayer in a standard block
3. Keep residual connections and layer norm unchanged

Output:
```python
class CAttnU(nn.Module):
    """Continued Fraction Attention (Univariate) with causal masking."""

    def __init__(self, embed_dim, seq_len, num_ladders=8, depth=5):
        super().__init__()
        self.seq_len = seq_len
        self.num_ladders = num_ladders
        # Two ensembles for cross-term interaction
        self.ladders_a = nn.ModuleList([
            Ladder(seq_len, depth) for _ in range(num_ladders)
        ])
        self.ladders_b = nn.ModuleList([
            Ladder(seq_len, depth) for _ in range(num_ladders)
        ])
        # Upper-triangular mask for causality after transpose
        mask = torch.triu(torch.ones(embed_dim, seq_len), diagonal=0)
        self.register_buffer('causal_mask', mask.unsqueeze(0))  # (1, embed_dim, seq_len)
        self.out_proj = nn.Linear(embed_dim, embed_dim)

    def forward(self, x):
        # x: (batch, seq_len, embed_dim)
        # Transpose to (batch, embed_dim, seq_len) for univariate ladders
        xt = x.transpose(1, 2)
        xt = xt * self.causal_mask  # enforce causality

        out_a = torch.stack([l(xt) for l in self.ladders_a], dim=0).mean(0)
        out_b = torch.stack([l(xt) for l in self.ladders_b], dim=0).mean(0)

        # Element-wise product creates interaction terms
        combined = out_a * out_b  # (batch, embed_dim, seq_len)
        combined = combined.transpose(1, 2)  # back to (batch, seq_len, embed_dim)
        return self.out_proj(combined)


class CoFrGeNetBlock(nn.Module):
    """Transformer block with CoFrGeNet components replacing MHA and FFN."""

    def __init__(self, embed_dim, seq_len, num_ladders=8, depth=5):
        super().__init__()
        self.ln1 = nn.LayerNorm(embed_dim)
        self.attn = CAttnU(embed_dim, seq_len, num_ladders, depth)
        self.ln2 = nn.LayerNorm(embed_dim)
        self.ffn = CFFN(embed_dim, num_ladders, depth)

    def forward(self, x):
        x = x + self.attn(self.ln1(x))
        x = x + self.ffn(self.ln2(x))
        return x
```

## Best Practices

- **Do:** Clamp `K_d` away from zero with epsilon (0.01 works well) before division. Poles at zero are the primary numerical hazard in continued fractions.
- **Do:** Use the custom backward pass with analytical gradients rather than relying on autograd through nested reciprocals. This prevents gradient explosion at near-pole values.
- **Do:** Apply the dyadic training schedule -- train deeper ladder levels for exponentially fewer steps. Start with the linear term, then progressively unfreeze deeper depths.
- **Do:** Start with small depth (d=3 or d=5) and moderate ensemble size (L=8). Only increase if validation loss plateaus.
- **Avoid:** Using large depth values (d > 7) without the dyadic schedule. Deep ladders without staged training are unstable and prone to divergence.
- **Avoid:** Removing the gating mechanism in CFFN. The sigmoid gate controls information flow into the ladders and is essential for stable training.
- **Avoid:** Applying batch normalization inside ladders. Layer norm before the block is sufficient; internal normalization disrupts the continued-fraction computation.

## Error Handling

| Problem | Symptom | Fix |
|---------|---------|-----|
| Division by zero / NaN loss | `K_d` passes through zero during training | Increase epsilon clamp value (try 0.05); reduce learning rate |
| Gradient explosion | Loss spikes suddenly after stable training | Enable gradient clipping (max norm 1.0); verify custom backward is used instead of autograd |
| Poor convergence at high depth | Validation loss plateaus early | Apply dyadic schedule; verify deeper weights are frozen initially |
| Causality violation in CAttnU | Future tokens influence past predictions | Verify upper-triangular mask is applied after transpose; check mask dimensions match `(embed_dim, seq_len)` |
| Memory issues with large L | OOM during ladder ensemble forward | Reduce `num_ladders`; compute ladders sequentially with gradient checkpointing instead of stacking |

## Limitations

- **Fixed sequence length in CAttnU.** The upper-triangular causal mask and transposed ladder dimensions are tied to a specific sequence length. Variable-length inputs require padding or re-initialization of the mask. CAttnM is more flexible for variable-length scenarios.
- **Not a universal drop-in for all architectures.** CoFrGeNet components have been validated on GPT-2-xl and Llama-3 scales. Behavior at very large scales (>10B parameters) or on non-autoregressive tasks (e.g., encoder-only BERT-style models) is not established by the paper.
- **Hyperparameter sensitivity.** The depth `d`, ensemble size `L`, and epsilon clamp interact. Tuning these requires experimentation -- there is no single universal setting.
- **Custom gradients require maintenance.** The analytical backward pass must be kept in sync with any architectural changes to the forward pass. Standard autograd can be used as a fallback for prototyping (at the cost of numerical precision).
- **Training schedule complexity.** The dyadic schedule adds implementation overhead compared to standard constant-LR or cosine-decay schedules. It is important but non-trivial to implement correctly with modern training frameworks.

## Reference

**Paper:** [CoFrGeNet: Continued Fraction Architectures for Language Generation](https://arxiv.org/abs/2601.21766v2) (Dhurandhar et al., 2026)

Look for: Section 3 (Methodology) for the continuant recurrence and gradient derivations; Section 3.2-3.3 for CAttnU/CAttnM/CFFN architectures; Section 4 for the dyadic training schedule; Tables 1-3 for parameter counts and perplexity comparisons against GPT-2-xl and Llama-3 baselines.